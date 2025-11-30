---
title: "Golang design: Mechanics of Coroutines"
author: "Vadim Inshakov"
---


![](https://cdn-images-1.medium.com/max/2088/1*5_sw6i26kK7Js2Ag1HjLGw.png)
>  A step-by-step breakdown of how coroutines work in Go, explained simply

While coroutines haven’t been added to Go yet, **Russ Cox** [has presented](https://research.swtch.com/coro) an implementation that requires no changes to the language, allowing you to use them right now.

We’ll look at a basic version without cancellations or panic handling mechanisms to understand the core concept of how coroutines work.

Here’s the coroutine implementation by **Russ Cox**:

```go
func NewCoroutine[In, Out any](f func(in In, yield func(Out) In) Out) (resume func(In) Out) {
    cin := make(chan In)
    cout := make(chan Out)
        
    resume = func(in In) Out {
        cin <- in
        return <-cout
    }
        
    yield := func(out Out) In {
        cout <- out
        return <-cin
    }
        
    go func() {
        cout <- f(<-cin, yield)
    }()
        
    return resume
}
```

If you, like me, find this difficult to understand or if it doesn’t easily fit into your mental model, let’s break it down and understand how it works **step by step**.

## Analyzing the Behavior

To begin, let’s write a simple test to see how this code behaves:

```go
func Test_coroutine_simple(t *testing.T) {
 resume := NewCoroutine(func(in int, yield func(str string) int) string {
  for {
   fmt.Println("before yield", in)
   in = yield("out of yield: " + strconv.Itoa(in))
   fmt.Println("after yield", in)
  }
    
  return "end coro"
 })
    
 fmt.Println("first resume output:", resume(0))
 fmt.Println("second resume output:", resume(1))
}
```

Here’s the output:

```go
before yield 0
first resume output: out of yield: 0
after yield 1
before yield 1
second resume output: out of yield: 1
```

## How Does It Work?

Let’s break it down step by step:

1. **NewCoroutine returns the resume function**, which allows us to start or resume the coroutine (though we haven’t started it yet).

2. We call resume with an argument: resume(0).

3. The **coroutine’s body** starts executing. Here’s a reminder of what the body looks like:

   for {
   fmt.Println("before yield", in)
   in = yield("out of yield: " + strconv.Itoa(in)) // Stop here
   fmt.Println("after yield", in)
   }

4. The loop begins, prints before yield 0, calls yield, and **stops**.

5. The yield function sends "out of yield: 0" back to the caller.

6. The execution pauses at fmt.Println("first resume output:", resume(0)).

7. The main program prints: first resume output: out of yield: 0.

8. We call resume(1) again, and the **coroutine resumes** execution.

9. It prints after yield 1, then starts the next iteration of the loop:

   for {
   fmt.Println("before yield", in)
   in = yield("out of yield: " + strconv.Itoa(in))
   fmt.Println("after yield", in)
   }

10. It prints before yield 1 and **yields** again, returning "out of yield: 1" to the caller.

11. The main program prints: second resume output: out of yield: 1.

## Visual Representation

Here’s a simple flowchart of what’s happening:

```go
Start
                             |
                             V
            +--------------------------------+
            | NewCoroutine returns resume()  |  <-- creates the `resume` function
            +--------------------------------+
                             |
                             V
                   +--------------------+
                   | Call resume(0)     |  <-- first coroutine call: `resume(0)`
                   +--------------------+
                             |
                             V
           +--------------------------------------+
           | Coroutine starts executing           |  <-- coroutine starts running
           | Print: "before yield 0"              |  
           | in = yield("out of yield: 0")        |  <-- calls `yield`
           +--------------------------------------+
                             |
                             V
         +--------------------------------------+
         | yield sends "out of yield: 0"       |  <-- coroutine sends the result via channel
         | and waits for input                 |  <-- coroutine is paused, waiting for input
         +--------------------------------------+
                             |
                             V
                  +-------------------------+
                  | Execution stops          |
                  | Main prints:             |
                  | "first resume output:    |
                  | out of yield: 0"         |  <-- main thread prints the first output
                  +-------------------------+
                             |
                             V
                 +------------------------+
                 | Call resume(1)         |  <-- second coroutine call: `resume(1)`
                 +------------------------+
                             |
                             V
          +--------------------------------------+
          | Coroutine resumes                   |  <-- coroutine resumes
          | yield receives 1                    |  <-- `yield` receives input (1)
          | Print: "after yield 1"              |  
          +--------------------------------------+
                             |
                             V
          +--------------------------------------+
          | Next iteration of the loop:          |
          | Print: "before yield 1"              |  
          | in = yield("out of yield: 1")        |  <-- calls `yield` again, pauses here
          +--------------------------------------+
                             |
                             V
         +--------------------------------------+
         | yield sends "out of yield: 1"       |  <-- coroutine sends another result via channel
         | and waits for input                 |  <-- coroutine is paused again, waiting for input
         +--------------------------------------+
                             |
                             V
                  +-------------------------+
                  | Execution stops          |
                  | Main prints:             |
                  | "second resume output:   |
                  | out of yield: 1"         |  <-- main thread prints the second output
                  +-------------------------+
```

Go back to the Test_coroutine_simple code and re-read it with this new understanding. If you have any questions, feel free to ask in the comments below!

## Internal Mechanics of Coroutines

Now, let’s explore **how the coroutine mechanism works**.

First, **two channels** are created:

```go
cin := make(chan In)
cout := make(chan Out)
```

* cin is used to send input values to the coroutine.

* cout is used to send output values back from the coroutine.

**resume Function:**

```go
resume = func(in In) Out {
   cin <- in
   return <-cout
}
```

* The resume function sends an input value to the coroutine through cin.

* Then it waits for the coroutine to return an output via cout.

**Coroutine:**

```go
go func() {
    cout <- f(<-cin, yield)
}()
```

* A **goroutine** is started, running the coroutine in a separate light thread.

* The coroutine function f receives its first input via cin, then uses the yield function to send back intermediate results via cout.

## Why cout <- f(<-cin, yield)?

You might wonder why we return a value to cout in the goroutine, even though yield does this already.

The reason is that this final cout <- f(...) is needed to return the **last value** after the coroutine finishes. In the previous test, this would be "end coro" if the function reaches a return.

This concludes the technical breakdown of coroutines in Go. I hope this helped make the concept clearer for you.

*If you’re interested in contributing to exciting open-source projects,[ feel free to join](https://github.com/vadiminshakov?tab=repositories)*
