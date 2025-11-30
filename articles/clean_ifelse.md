---
title: "Clean Code Fundamentals: Eliminating if/else hell using functions"
author: "Vadim Inshakov"
---


### Why you need to stop using if/else

![](https://cdn-images-1.medium.com/max/2048/1*pFlDV203xLENN7hGuiFckA.png)

You’re probably familiar with the [early return pattern](https://danp.net/posts/reducing-go-nesting/). However, it’s not always clear how to apply it. Often, you find yourself writing a sequence of instructions in such a way that each subsequent one should only be executed if the previous one completed without errors:

```go
func calculateSomething(x, y int){
{ // not the main block inside the function, just logging the call arguments to a log file
  f, err := os.OpenFile("calculateSomething.log", os.O_CREATE|os.O_RDWR|os.O_TRUNC, l.opts.FilePerms)
  if err == nil { // enter here only if there is no error
     defer f.Close()
     if _, err := f.Write(fmt.Sprintf("%d, %d", x, y)); err == nil { 
         if err := f.Sync(); err != nil { // execute only if there was no previous error
           log.Printf("failed to fsync file: %s", err)
         }
     } else {
      log.Printf("failed to write file: %s", err)
     }
         
     return f.Close()
 } else {
    log.Printf("failed to open file: %s", err)
 }
}
     
other instructions...
}
```

In this case, you can’t perform an early return, as the function needs to complete regardless of the file-related block. You just want to log the file-related error and *continue execution*. Early return won’t work here.

The first thing that comes to mind — can you extract the problematic block into a separate function?

```go
func logCalc(x, y int) error {
  f, err := os.OpenFile("calculateSomething.log", os.O_CREATE|os.O_RDWR|os.O_TRUNC, l.opts.FilePerms)
  if err != nil {
    return fmt.Errorf("failed to open file: %w", err)
  }
  defer f.Close()
    
  if _, err := f.Write(fmt.Sprintf("%d, %d", x, y)); err != nil { 
    return fmt.Errorf("failed to write file: %w", err)
  } 
  if err := f.Sync(); err != nil {
    return fmt.Errorf("failed to fsync file: %w", err)
  }
         
  return nil
}
```

Now you can confidently return from any point within the auxiliary function without the need to create unnecessary nesting.

Now the main function looks like this:

```go
func calculateSomething(x, y int){
  if err := logCalc(x, y); err != nil {
    log.Printf("file manipulation error: %s", err)
  }
     
  other instructions...
}
```

It’s better, isn’t it?

But what if the `logCalc` function is not used anywhere else? There’s no need to separate into a function something that is not used elsewhere (exceptions are only cases where the top-level function is too large and can be divided for readability improvements). We can rewrite our solution like this:

```go
func calculateSomething(x, y int){
// auxiliary anonymous function
if err := func(x, y int) error {
  f, err := os.OpenFile("calculateSomething.log", os.O_CREATE|os.O_RDWR|os.O_TRUNC, l.opts.FilePerms)
  if err != nil {
    return fmt.Errorf("failed to open file: %w", err)
  }
  defer f.Close()
    
  if _, err := f.Write(fmt.Sprintf("%d, %d", x, y)); err != nil { 
    return fmt.Errorf("failed to write file: %w", err)
  } 
  if err := f.Sync(); err != nil {
    return fmt.Errorf("failed to fsync file: %w", err)
  }
         
  return nil
}(); err != nil {
    log.Printf("file manipulation error: %s", err)
}
    
// main functionality of the function
...
}
```

We made the `logCalc` function anonymous and embedded it into calculateSomething. The function remains just as clean without if/else hell, and at the same time, we eliminated the unnecessary function that was not used anywhere and could confuse other project participants.
