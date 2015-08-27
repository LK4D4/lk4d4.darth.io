+++
title = "Mystery of finalizers in Go"
description = "How to write finalizers for objects and why you should avoid them"
date = "2015-08-26"
categories = ["go"]
+++

## Finalizers

Finalizer is basically a function which will be called when your object will lost
all references and will be found by GC. In Go you can add finalizers to your
objects with [`runtime.SetFinalizer`](https://golang.org/pkg/runtime/#SetFinalizer)
function. Let's see how it works.
{{< highlight go >}}
package main

import (
    "fmt"
    "runtime"
    "time"
)

type Test struct {
    A int
}

func test() {
    // create pointer
    a := &Test{}
    // add finalizer which just prints
    runtime.SetFinalizer(a, func(a *Test) { fmt.Println("I AM DEAD") })
}

func main() {
    test()
    // run garbage collection
    runtime.GC()
    // sleep to switch to finalizer goroutine
    time.Sleep(1 * time.Millisecond)
}
{{< /highlight >}}
Output obviously will be:
```
I AM DEAD
```
So, we created object `a` which is pointer and set simple finalizer to it. When
code left test function - all references to it disappeared and therefore garbage
collector was able to collect `a` and call finalizer in its own goroutine. You
can try to modify `test()` function to return `*Test` an print it in `main()`,
then you'll see that finalizer won't be called.
Also if you remove `A` field from `Test` type, because then `Test` became just
empty struct and empty struct allocates no memory and can't be collected by GC.

## Finalizers examples

Let's try to find finalizers usage in standard library. There it is used only for
only used for closing file descriptors like this in `net` package:
{{< highlight go >}}
runtime.SetFinalizer(fd, (*netFD).Close)
{{< /highlight >}}
So, you'll never leak fd even if you forget to `Close` `net.Conn`.

So probably finalizers not so good idea if even in standard library it has so
limited usage. Let's see what problems can be.

## Why you should avoid finalizers

Finalizers is pretty tempting idea if you come from languages without GC or where
you're not expecting users to write proper code. In Go we have both GC and
pro-users :) So, in my opinion explicit call of `Close` is always better than
magic finalizer. For example there is finalizer for fd in `os`:
{{< highlight go >}}
// NewFile returns a new File with the given file descriptor and name.
func NewFile(fd uintptr, name string) *File {
    fdi := int(fd)
    if fdi < 0 {
        return nil
    }
    f := &File{&file{fd: fdi, name: name}}
    runtime.SetFinalizer(f.file, (*file).close)
    return f
}
{{< /highlight >}}
and `NewFile` is called by `OpenFile` which is called by `Open`, so if you're
opening file you'll hit that code. Problem with finalizers that you have no
control over them, and more than that you're not expecting them. Look at code:
{{< highlight go >}}
func getFd(path string) (int, error) {
    f, err := os.Open(path)
    if err != nil {
        return -1, err
    }
    return f.Fd(), nil
}
{{< /highlight >}}
It's pretty common operation to get file descriptor from path when you're
writing some stuff for linux. But that code is unreliable, because when you're
return from `getFd` `f` loses its last reference and so your file is doomed to
be closed sooner or later (when next GC cycle will come). Here is problem not
that file will be closed, but that it's not documented and not expected at all.

## Conclusion

I think it's better to suppose that users are smart enough to cleanup object
themselves. At least all methods which call `SetFinalizer` should document this,
but I personally don't see any value in this method for me.
