---
title: "Deep Dive into Go Runtime: Advanced Thread Management Explained"
author: "Vadim Inshakov"
---

> Unveiling the hidden aspects of thread management in Go.

![](https://miro.medium.com/v2/resize:fit:1400/1*wW2W5fD8rgOz8U2sYqwHOQ.png)

We all know that creating more threads than there are CPU cores on a machine is pointless. A core can only execute one task at a time. Adding more threads than cores only increases the load on the OS scheduler. Basic knowledge, yes, everyone knows that.

But what if I told you that having 300 threads in reserve for your Go application is beneficial? No, it’s not madness; it’s about understanding the runtime’s peculiarities.

Here’s the revised and corrected version of the text with more precision:

Let’s take a step back and recall `netpoll`—a Go runtime abstraction for managing network events efficiently. The task of `netpoll` is to monitor file descriptors registered with the operating system for network events. These file descriptors represent sockets through which we expect responses or intend to send requests. `netpoll` delegates event detection to the operating system using mechanisms like `epoll` (Linux), `kqueue` (macOS/BSD), or IOCP (Windows). This allows us to avoid blocking threads while waiting for network events. Instead, threads in Go continue executing other goroutines.

A goroutine performing an IO operation gets parked (taken off scheduling) if data is not immediately available, or if the system call cannot be performed at the moment. The runtime marks the goroutine as waiting and registers the corresponding file descriptor in the `netpoll` mechanism:

[https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/netpoll.go#L573](https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/netpoll.go#L564)

```go
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool { gpp := &pd.rg  ... lines skipped if waitio || netpollcheckerr(pd, mode) == pollNoError {  gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceBlockNet, 5) } ... lines skipped}
```

When a response arrives or the requested event occurs, the operating system notifies the Go runtime. The runtime identifies the goroutines that were waiting on the event and places them into the local run queue of a `P` (a logical processor in Go's runtime). These goroutines are then scheduled for execution.

Here’s an example of how `netpoll` returns a list of goroutines that have been waiting for events. The returned list is added to the execution queue of the runtime for processing:

[https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L3206](https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L3206)

```go
func findRunnable() (gp *g, inheritTime, tryWakeP bool) { // Poll network until next timer. if netpollinited() && (netpollWaiters.Load() > 0 || pollUntil != 0) && sched.lastpoll.Swap(0) != 0 {  ... lines skipped  list := netpoll(delay) // block until new work is available    ... lines skipped  lock(&sched.lock)  pp, _ := pidleget(now)  unlock(&sched.lock)  if pp == nil {   injectglist(&list) <---- !  } else {    ... lines skipped}
```

Here’s the revised and clarified version of your text:

**Intermediate Conclusion:** The Go runtime doesn’t require additional threads to handle network I/O because this work is delegated to OS-specific I/O subsystems (e.g., `epoll` on Linux, `kqueue` on Darwin/FreeBSD, and I/O completion ports on Windows) while goroutines that are waiting for network events are parked (taken off scheduling). This allows the runtime to efficiently manage resources and avoid blocking threads.

However, this approach works only for network requests. If you perform a system call unrelated to networking, such as reading or writing to a file, the asynchronous `netpoll`**mechanism cannot be used**. In this case**, the thread performing the operation will be blocked until the system call completes**.

So, how does the Go runtime handle this situation? When a blocking system call is made, the thread executing it becomes blocked, and the runtime adapts by creating a new thread to continue serving the context of the “sleeping” thread. This ensures that other goroutines can continue executing without being impacted by the blocking operation.

It’s important to note that this is a high-level overview of how the Go runtime manages I/O operations. The actual implementation is highly optimized and platform-specific, designed to balance performance and resource efficiency.

[https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#](https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L3206)L2649

```go
func handoffp(pp *p) { // handoffp must start an M in any situation where // findrunnable would return a G to run on pp. // if it has local work, start it straight away if !runqempty(pp) || sched.runqsize != 0 {  startm(pp, false, false)  return}... lines skipped}
```

**Please note:** system calls that cannot be handled using netpoll will spawn new OS threads if the runtime has no available threads. If there are already available threads and the Go runtime is aware of them, they will be reused.

[https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#](https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L3206)L2563

```go
func startm(pp *p, spinning, lockheld bool) { ... lines skipped nmp := mget() <---- get thread if exists if nmp == nil {  id := mReserveID()  unlock(&sched.lock)  var fn func()  if spinning {   // The caller incremented nmspinning, so set m.spinning in the new M.   fn = mspinning  }  newm(fn, pp, id) <--- create a new thread if there is none free... lines skipped}
```

This means that the more threads the runtime has, the easier it is to perform system calls. The number of CPU cores doesn’t matter in this case because it’s not about computation but rather IO work.

However, there’s an important caveat: no matter how many threads you have, they can’t do any useful work for the runtime if there are no available `P`’s. `P` is a logical context for `M` (the OS thread), and `M` can’t work without a `P`. The more P’s you create, the more you can leverage OS threads to perform tasks. The number of `P`’s is determined by the `GOMAXPROCS` variable (or `runtime.GOMAXPROCS()`). This trick was used in Dgraph:

[https://groups.google.com/g/golang-nuts/c/jPb_h3TvlKE](https://groups.google.com/g/golang-nuts/c/jPb_h3TvlKE)

## Proof

I couldn’t come up with simpler system calls to use for benchmarks other than basic file read and write operations. What could be simpler? Let’s write a benchmark in which, on each iteration, we launch 100 goroutines, each of which opens a file, writes to it, and reads from it.

```go
func BenchmarkX(b *testing.B) { runtime.GOMAXPROCS(8) os.Mkdir("./testlogdata", 0755) for i := 0; i < b.N; i++ {  var g errgroup.Group  for j := 0; j < 100; j++ {   g.Go(func() error {    suffix := strconv.Itoa(j)    f, err := os.OpenFile("./testlogdata/ff"+suffix, os.O_CREATE|os.O_TRUNC|os.O_RDWR, 0755)    require.NoError(b, err)    f.WriteString("hello")    f.Close()    os.Remove("./testlogdata/ff" + suffix)    return nil   })  }  g.Wait() } os.RemoveAll("./testlogdata")}
```

The second benchmark will test the same case but with a pre-set `GOMAXPROCS=200`.

```go
func BenchmarkY(b *testing.B) { runtime.GOMAXPROCS(200) os.Mkdir("./testlogdata", 0755) for i := 0; i < b.N; i++ {  var g errgroup.Group  for j := 0; j < 100; j++ {   g.Go(func() error {    suffix := strconv.Itoa(j)    f, err := os.OpenFile("./testlogdata/ff"+suffix, os.O_CREATE|os.O_TRUNC|os.O_RDWR, 0755)    require.NoError(b, err)    f.WriteString("hello")    f.Close()    os.Remove("./testlogdata/ff" + suffix)    return nil   })  }  g.Wait() } os.RemoveAll("./testlogdata")}
```

Results:

```go
go test . -run=xxx -bench=. -benchtime=7sgoos: darwingoarch: amd64cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHzBenchmarkX-12               1280           6180366 ns/optesting: BenchmarkX-12 left GOMAXPROCS set to 8BenchmarkY-12               1776           4738673 ns/optesting: BenchmarkY-12 left GOMAXPROCS set to 200
```

The version with an excess of pre-created `P`’s performed **38.75%** more operations (1776 operations) than the same code with `GOMAXPROCS` set to the number of CPU cores. In reality, both versions create the same number of threads, creating a new thread for each system call.

So, what’s the difference? The difference lies in the fact that in the second benchmark, we create and initialize the `P` pool before starting the work, more so than in the first benchmark.

Here, you can see how the runtime initializes the number of `P`’s to be equal to `GOMAXPROCS`:

[https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L1471](https://github.com/golang/go/blob/release-branch.go1.21/src/runtime/proc.go#L1471)

```go
for p1 != nil {  p := p1  p1 = p1.link.ptr()  if p.m != 0 {   mp := p.m.ptr()   p.m = 0   if mp.nextp != 0 {    throw("startTheWorld: inconsistent mp->nextp")   }   mp.nextp.set(p)   notewakeup(&mp.park)  } else {   // Start M to run P.  Do not start another M below.   newm(nil, p, -1) <--- start new thread  } }
```

**What happens if the runtime creates too many threads?**

Unused threads get “parked.” Threads can go to sleep immediately after starting or at any point during program execution if there is no work for them. However, if you exceed the limit of 10,000 OS threads, the program will panic with a message.

```go
runtime: program exceeds 10000-thread limitfatal error: thread exhaustion.
```

You can increase the allowed number of threads using `debug.SetMaxThreads(N)`, but I do not recommend doing so.

**Conclusions:**

- A large number of threads can be beneficial if a program makes many system calls.
- Threads are meaningless without P, the quantity of which you can configure using GOMAXPROCS.
- Network I/O doesn’t require additional threads and is performed using the asynchronous I/O facilities provided by the operating system.
