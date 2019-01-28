+++
title = "Unprivileged containers in Go, Part2: UTS namespace (setup namespaces)"
description = "How to setup namespaces before process execution"
date = "2015-07-17"
categories = ["go", "linux", "namespaces", "containers"]
+++

## Setup namespaces

In previous part we created some namespaces and executed process in them. It was
cool, but in real world we need to setup namespaces before process starts.
For example setup mounts, make chroot, set hostname, create network interfaces
etc. We need this because we can't expect from user process that it will do it,
it want all ready to execute.

So, in our case we want to insert some code after namespaces creation, but before
process execution. In C it's pretty easy to do, because there is `clone` call
there. Not so easy in Go(but easy, really). In Go we need to spawn new process
with our code in new namespaces. We can do it with executing our own binary
again with different arguments.

Look at code:

{{< highlight go >}}
    cmd := &exec.Cmd{
        Path: os.Args[0],
        Args: append([]string{"unc-fork"}, os.Args[1:]...),
    }
{{< /highlight >}}

Here we create `*exec.Cmd` which will call same binary with same arguments as
caller, but will replace `os.Args[0]` with string `unc-fork` (yes, you can
specify any `os.Args[0]`, not only program name).
It will be our keyword, which indicates that we want to setup namespaces and
execute process.

So, let's insert at the top of `main()` function next lines:

{{< highlight go >}}
    if os.Args[0] == "unc-fork" {
        if err := fork(); err != nil {
            log.Fatalf("Fork error: %v", err)
        }
        os.Exit(0)
    }
{{< /highlight >}}

It means next: execute function `fork()` and exit in special case of
`os.Args[0]`.

Let's write `fork()` now:

{{< highlight go >}}
func fork() error {
    fmt.Println("Start fork")
    path, err := exec.LookPath(os.Args[1])
    if err != nil {
        return fmt.Errorf("LookPath: %v", err)
    }
    fmt.Printf("Execute %s\n", append([]string{path}, os.Args[2:]...))
    return syscall.Exec(path, os.Args[1:], os.Environ())
}
{{< /highlight >}}

It's simplest `fork()` function, it's just prints some messages before starting
process. Let's look at `os.Args` array here. For example if we wanted to spawn
`sh -c "echo hello"` in namespaces, then now array looks like
`["fork", "sh", "-c", "echo hello"]`. We resolving `"sh"` as `"/bin/sh"` and call
{{< highlight go >}}
syscall.Exec("/bin/sh", []string{"sh", "-c", "echo hello"}, os.Environ())
{{< /highlight >}}

`syscall.Exec` calls `execve` syscall, you can read about it more in
`man execve`. It receives path to binary, arguments and array of environmental
variables. Here we just passing all variables down to process, but we can change
them in `fork()` too.

## UTS namespace

Let's do some real work in our new shiny function. Let's try to setup hostname
for our "container" (by default it inherits hostname of host). Let's add next
lines to `fork()`:

{{< highlight go >}}
    if err := syscall.Sethostname([]byte("unc")); err != nil {
        return fmt.Errorf("Sethostname: %v", err)
    }
{{< /highlight >}}

If we try to execute this code we'll get:
```
Fork error: Sethostname: operation not permitted
```

because we're trying to change hostname in host's UTS namespace.

From `man namespaces`:
```
UTS  namespaces  provide  isolation  of two system identifiers: the hostname and the NIS domain name.
```
So let's isolate our hostname from host's hostname. We can create our own UTS
namespace by adding `syscall.CLONE_NEWUTS` to `Cloneflags`. Now we'll see
successfully changed hostname:

```
$ unc hostname
unc
```
## Code

Tag on github for this article is `uts_setup`, it can be found
[here](https://github.com/LK4D4/unc/tree/uts_setup). I added some functions
to separate steps, created `Cfg` structure in `container.go` file, so later we
can change container configuration in one place.
Also I added logging with awesome library
[logrus](https://github.com/sirupsen/logrus).

Thanks for reading! I hope to see you next week in part about mount namespaces,
it'll be very interesting.

Previous parts:

 * [Part 1](/posts/unpriv1)
