---
title: "Golang design: Generics vs Interfaces, How It Really Works Under the Hood"
author: "Vadim Inshakov"
---

>  A long story in simple words about how the internals of Go work when using generics and interfaces. Be the better engineer by making right decisions.

![](https://cdn-images-1.medium.com/max/2048/1*uT60RAQxOBNCObcd_ZRLgw.png)

Sometimes you just need to have a function or method that accepts arguments with the same behavior. What to use: generics or interfaces?

The answer requires a deep dive into the Go’s design.

**WARN! Spoiler:** _in most cases, it’s better to use interfaces, but there are cases where generics would be more optimal._

Let’s look at two examples of polymorphism.

```go
type counter interface {
 Add()
}
    
type counterImpl1 struct {
 n int
}
    
func (p counterImpl1) Add() {
 p.n++
}
    
func add(p counter) {
 p.Add()
}
```

The function add takes counterImpl1, which satisfies the counter interface and calls its method. We can do the same thing using generics:

```go
func addGeneric[T counter](p T) {
 p.Add()
}
```

This works just like the previous example. We simply created two identical functions with a parameter that has specific methods. Is there any difference? Yes, there is.

### Dynamic dispatch

Interfaces are based on dynamic dispatch. This means that when calling an interface method, the go runtime has to find the address of the method (a function taking an object of the type on which the method is defined) of the specific type implementing the interface. The compiler creates tables that match calls of interface methods with specific methods, with the specific methods represented as pointers. Go runtime uses these tables to get a pointer to the required method (actually a function pointer), which is dereferenced and then called. More about the structure of interfaces can be found [here](https://research.swtch.com/interfaces) and [here](https://www.tapirgames.com/blog/golang-interface-implementation).

Thus, calling any interface method involves dereferencing. This is problematic because it doesn’t allow efficient use of the processor’s cache. We discussed this in more detail [in the previous article](https://medium.com/stackademic/go-when-pointers-reduce-the-impact-of-cpu-caching-1e290fc30077).

Generics in Go operate based on hybrid (partial) monomorphization. What is monomorphization? Let’s find out.

Suppose you have a generic function:

```go
func add[T int | string](x, y T) T {
  return x + y
}
```

When you compile this code, you’ll get something like this in the resulting binary (pseudocode):

```go
func addInt(x, y int) int {
  return x + y
}
    
func addString(x, y string) string {
  return x + y
}
```

This is monomorphization: the compiler automatically generates multiple copies of your types, functions, and methods for each usage scenario. This means zero cost for the runtime, and this is mainly how generics work in compiled languages.

Go uses a similar technique called “GCShape stenciling with Dictionaries”. Stenciling is just a rephrased “monomorphization”. And GCShape refers to groups of types that can be interchangeably used when instantiating generic objects. “Two concrete types are in the same gcshape grouping if and only if they have the same underlying type or they are both pointer types” — this is how it’s described [in the design document](https://github.com/golang/proposal/blob/master/design/generics-implementation-dictionaries-go1.18.md) of Go generics.

This means that if we instantiate our add function for the type type Key string and for the type type Name string, we only need to create one version of the add function during compilation because these types form one GCShape. And that’s great. But there’s a downside: all pointers form a single GCShape, which necessitates passing dictionaries to the resulting function, which contain lists of methods of those types the pointers refer to (for the same dynamic dispatch we discussed earlier).

This means that a function taking an interface and a generic **function *both use dynamic dispatch***. Only generic functions need to first retrieve type information from the dictionary, and then find its method, which adds some overhead.

### Generic vs Interface

Let’s check how both implementations work.

We’ll create an interface and two functions (a regular one and a generic one) that work with this interface:

```go
type counter interface {
 Add()
 Sub()
 Multiply()
}
    
func addsubmul(p counter) {
 p.Add()
 p.Multiply()
 p.Sub()
}
    
func addsubmulGeneric[T counter](p T) {
 p.Add()
 p.Multiply()
 p.Sub()
}
```

Now let’s write benchmarks:

```go
func BenchmarkIface(b *testing.B) {
 b.Run("ifaceByPointer", func(b *testing.B) {
  impl1 := &counterImpl1Ptr{}
  impl2 := &counterImpl2Ptr{}
  for i := 0; i < b.N; i++ {
   for j := 0; j < 100; j++ {
    addsubmul(impl1)
    addsubmul(impl2)
   }
  }
 })
}
    
func BenchmarkGeneric(b *testing.B) {
 b.Run("genericByPointer", func(b *testing.B) {
  impl1 := &counterImpl1Ptr{}
  impl2 := &counterImpl2Ptr{}
  for i := 0; i < b.N; i++ {
   for j := 0; j < 100; j++ {
    addsubmulGeneric[*counterImpl1Ptr](impl1)
    addsubmulGeneric[*counterImpl2Ptr](impl2)
   }
  }
 })
}
```

*(I added an extra 100 iterations inside the benchmark loop to get more illustrative results)*

The results are roughly the same, as we expected:

```go
go test --bench=. --benchtime=10s
    
goos: darwin
goarch: amd64
pkg: module05/fibonachi
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkIface/ifaceByPointer-12        11261798               986.5 ns/op
BenchmarkGeneric/genericByPointer-12    11983582               996.4 ns/op
```

### Generics are faster when passing arguments by value

I made a small main function where we initialize a structure and pass it by value to a function that takes an interface (line 39), and to a generic function.

![](https://cdn-images-1.medium.com/max/4784/1*R-r0MOyZH9CZ9JhACMimMg.png)

We see that a dictionary is being used here! But for what purpose?

![](https://cdn-images-1.medium.com/max/2472/1*XNoVuu9ocsUVZ-MFVVmqyA.png)

The dictionary is passed to the AX register.

![](https://cdn-images-1.medium.com/max/3136/1*WuKg1sd14906q-R12UZkDg.png)

(1) deref method pointer from dictionary and pass it to `CX` register;

(2) call method

This is assembly for:

```go
func addsubmulGeneric[T counter](p T) {
 p.Add() <--
 ...
}
```

And then it just repeats. For `p.Multiply()` we see this:

![](https://cdn-images-1.medium.com/max/2000/1*d0EuQE3vaYl8O8sCpM2ojQ.png)

(1) take dictionary address from stack;

(2) deref dict address + 8 bytes offset (method Multiply);

(3) call method

Even though we passed not a pointer but a structure by value to the generic function, and ***it seems that the compiler could monomorphize this code, but it still uses dictionaries***, and this should negatively impact performance.

Benchmark:

```go
func BenchmarkIface(b *testing.B) {
 b.Run("ifaceByValue", func(b *testing.B) {
  impl1 := counterImpl1{}
  impl2 := counterImpl2{}
  for i := 0; i < b.N; i++ {
   for j := 0; j < 100; j++ {
    addsubmul(impl1)
    addsubmul(impl2)
   }
  }
 })
}
    
func BenchmarkGeneric(b *testing.B) {
 b.Run("genericByValue", func(b *testing.B) {
  impl1 := counterImpl1{}
  impl2 := counterImpl2{}
  for i := 0; i < b.N; i++ {
   for j := 0; j < 100; j++ {
    addsubmulGeneric[counterImpl1](impl1)
    addsubmulGeneric[counterImpl2](impl2)
   }
  }
 })
}

go test --bench=. --benchtime=10s
    
goos: darwin
goarch: amd64
pkg: module5
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkIface/ifaceByValue-12           7997700              1382 ns/op
BenchmarkGeneric/genericByValue-12      13218322               933.3 ns/op
```

Incredibly, the generic function is faster, even though it uses the same dynamic dispatch! **The difference is 48%**.

Why?

![](https://cdn-images-1.medium.com/max/4720/1*WQ51SuzSpzbVDkSrtPwWRA.png)

* the compiler was able to inline the generic function

* arguments of a non-generic function are allocated on the heap

The thing is, when passing by value, Go knows the specific GCShape that the generic function accepts, knows its size, and can decide to allocate it on the stack because it’s much more efficient than the heap. When using interfaces, however, you don’t have a choice but to allocate on the heap, because an interface is a pointer type.

### Conclusion

Generics in Go are similar to implementations in other mainstream languages but use additional mechanisms (gcshape, dictionaries) to reduce binary sizes and compile times, *slightly sacrificing performance*. This may change in future releases of Go.

If you need a polymorphic function whose parameters are not of a pointer type, go with the generic implementation, as the compiler can better optimize it by avoiding heap allocations when possible.
