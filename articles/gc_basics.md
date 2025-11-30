---
title: "Basics of Golang GC Explained: Tri-color Mark and Sweep and Stop the World"
author: "Vadim Inshakov"
---


![](https://cdn-images-1.medium.com/max/2050/1*Uya5-_qK3mN0CuJRt_Ba_w.png)

I want to discuss with you some basics of garbage collection in Golang that are often elusive to understand. You know, during garbage collection, Go first (1) marks root objects as gray, (2) then marks them themselves as black, and their descendants as gray, and (3) finally removes the remaining (white) ones. This is known as mark and sweep. But why do we need three colors!? Why not two? Let’s discuss this further below, as well as determine the actual overhead from stop the world.

## Naive mark and sweep

If the entire task of GC boils down to marking objects that cannot be deleted and removing unmarked ones, why not use a two-color algorithm? Indeed, we can simply mark all reachable objects in black and then remove the white ones.

![](https://cdn-images-1.medium.com/max/2000/1*qatEZs_df4c31FXs9YWP_A.png)

It all seems logical, but now we’ve decided to make our garbage collector incremental, meaning we give it the ability to split the marking phase into several low latency stages.

![](https://cdn-images-1.medium.com/max/2000/1*77A5HnQxePnDvGuxtDBfDA.png)

As you can see, in the first stage, we mark node 1 as black first, then we look for connections of this node, find node 2, and mark it as black too. Then the marking pauses, giving a little CPU time to the application to perform its main task. And finally, we move on to the second stage of marking. Since there’s no distinction between gray/black, we don’t know if the references of node 1 have been checked, so we must check it again. The algorithm will likely eventually finish, but there’s no guarantee it will finish before the program itself completes (resulting in OOM).

To solve this problem, we add an invariant: **black objects should be considered scanned**.

It seems like the algorithm works and is able to terminate correctly even with incremental garbage collection. But there’s one catch that breaks everything: **throughout the entire operation of such a naive version of the mark and sweep algorithm, we have to keep the program in a stop the world state**.

## What would happen if mark and sweep were to operate without STW?

![](https://cdn-images-1.medium.com/max/2000/1*JOM7ORglf2byjOCtyXMSAA.png)

As you can see, between the two marking stages, many new objects have been added, but we don’t have a marker indicating that they need to be scanned. This leads to memory leaks. The only solution is to prohibit our program (mutator) from making any changes while the garbage collector is running.

## How does tricolor mark and sweep enable us to achieve concurrent garbage collection?

As we discovered earlier, we have an issue with stop the world (STW) in the mark and sweep algorithm. However, if we introduce an additional marker “unscanned but not eligible for deletion” (gray color), the STW problem is automatically resolved!

It should be noted that in Golang, GC is not just incremental but concurrent. The logic is the same as in the incremental approach — GC work is divided into separate parts, but they are executed not sequentially but concurrently by background workers.

So, here’s how tricolor marking looks in Golang:

![](https://cdn-images-1.medium.com/max/2000/1*NrJF-ki-mjVYSHUjFRgz9w.png)

We mark objects referenced by black nodes but not yet scanned as gray nodes. Let’s see how this helps us eliminate STW.

![](https://cdn-images-1.medium.com/max/2000/1*Z39EnS58f6jde4kHiYjB8Q.png)

Newly added objects are marked as gray and will be scanned by the gcBgMarkWorker2 or any other available worker. This doesn’t completely eliminate the need for STW but significantly reduces the time spent on it. At least we don’t need to stop the world to protect against heap changes during the marking phase. Next, we’ll delve into the specific stop the world pauses that still occur in Golang.

## “Stop the world” in Go is not a problem

Well, indeed, it’s not your concern in 99% of cases when you think about it. The main garbage collector work occurs concurrently with your program’s execution, and STW happens for a very short time between phases:

1. Transition from sweep to mark (sweep termination).

2. Transition from mark to sweep (mark termination).

In both cases, world stoppages occur in preparation for the next phase: starting workers, clearing P cache (processor in Go scheduler terms), waiting for the sweep phase to complete (for cases where you’ve triggered runtime.GC() from your program’s code alongside the runtime-initiated garbage collection), setting/unsetting the write barrier — and none of this is tied to the number of allocated objects.

If you allocated 1 megabyte and then started allocating gigabytes of objects, it doesn’t mean STW will grow proportionally because STW is only needed for a short time during the initialization of the next marking or sweeping phase. The actual marking and sweeping of this data array happens in the background.

The duration of STW depends on the number of involved P’s, the number of goroutines, and is associated with the need to manage their state. STW is not affected by the number of allocations on the heap.

The world stops only for two short periods: mark termination and sweep termination. Marking and sweeping occur in the background and do not block your application.

## Benchmark

I decided to throw together a simple test that tests two scenarios with the allocation of small objects. I disabled the garbage collector using env GOGC=off and simply trigger garbage collection at the end of each test by directly calling runtime.GC():

```go
func Test1000Allocs(t *testing.T) {
    go func() {
       for {
          i := 123
          reader(&i)
       }
    }()
    
    for i := 0; i < 1000; i++ {
       ii := i
       i = *reader(&ii)
    }
    
    runtime.GC()
}
    
func Test10000000000Allocs(t *testing.T) {
    go func() {
       for {
          i := 123
          reader(&i)
       }
    }()
    
    for i := 0; i < 10000000000; i++ {
       ii := i
       i = *reader(&ii)
    }
    
    runtime.GC()
}
    
//go:noinline
func reader(i *int) *int {
    ii := i
    return ii
}
```

We run each test with tracing information enabled:

```go
GOGC=off go test -run=Test1000Allocs -trace=trace1.out
GOGC=off go test -run=Test10000000000Allocs -trace=trace2.out
go tool trace trace1.out
go tool trace trace2.out
```

![Test1000Allocs](https://cdn-images-1.medium.com/max/6016/1*LgKvFfLLlwukRNbopA3u7A.png)

![Test1000Allocs](https://cdn-images-1.medium.com/max/6016/1*mGlyfingl0YD1_ANZg44Mg.png)

![Test10000000000Allocs](https://cdn-images-1.medium.com/max/6016/1*LgKvFfLLlwukRNbopA3u7A.png)

![Test10000000000Allocs](https://cdn-images-1.medium.com/max/6016/1*J9ysm9i9Pe9wX29rKG0dtQ.png)

80384 ns for STW in the sweep termination phase for the Test1000Allocs test. 88384 ns for Test10000000000Allocs. Is there a difference? I don’t see one either.

87616 ns for STW in the mark termination phase for the Test1000Allocs test. 120128 ns for Test10000000000Allocs. Here, the difference is more significant, but again, we’re talking about fractions of a millisecond. During most of the time while the GC was running, our program successfully ran in parallel.

Yes, with a large number of allocations and a low GOGC value, these short GC cycles with small STW phases can occur frequently and may be considered a problem. The test is somewhat synthetic because I disabled the GC and only triggered it once with a direct call. But this just means that sometimes it’s worth considering the chosen value of GOGC.

In conclusion, it’s evident that the garbage collector in Golang operates concurrently, and the pauses during stop the world in most cases are negligible issues.
