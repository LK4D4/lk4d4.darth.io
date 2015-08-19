+++
title = "Unprivileged containers in Go, Part1: User and PID namespaces"
description = "How to create basic container in Go, first steps"
date = "2015-07-15"
categories = ["go", "linux", "namespaces", "containers"]
+++

## Unprivileged namespaces

Unprivileged(or user) namespaces is
[Linux namespaces](http://man7.org/linux/man-pages/man7/namespaces.7.html), which
can be created from unprivileged(non-root) user. It is possible only with usage
of user namespaces. Exhaustive info about user namespaces you can find in
man-page `man user_namespaces`. Basically for creating your namespaces you need
to create user namespace first. Kernel can take job of creating namespaces in
right order for you, so you can just pass bunch of flags to `clone` and user
namespace always will be created first and will be `parent` for other namespaces.

## User namespace

In user namespace you can map user and groups from host to this namespace, so
for example your user with uid 1000 can be 0(root) in namespace.

Support for user and groups was
[introduced to Go](https://github.com/golang/go/commit/f9d7e139552b186f4c68a3a87b470847167a9076)
by Mrunal Patel and was released in 1.4.0. Unfortunately there was security fix
to linux kernel 3.18, which prevents group mappings from unprivileged user
without disabling `setgroups` syscall. It was
[fixed](https://github.com/golang/go/commit/f5c60ff2da4851f9056120a423ce6b48624fb97e)
by me and will be released in 1.5.0 (UPD: Already released!).

For executing process in new user namespace, you need to create `*exec.Cmd` like
this:

{{< highlight go >}}
cmd := exec.Command(binary, args...)
cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUSER
        UidMappings: []syscall.SysProcIDMap{
            {
                ContainerID: 0,
                HostID:      Uid,
                Size:        1,
            },
        },
        GidMappings: []syscall.SysProcIDMap{
            {
                ContainerID: 0,
                HostID:      Gid,
                Size:        1,
            },
        },
    }
{{< /highlight >}}

Here you can see `syscall.CLONE_NEWUSER` flag in `SysProcAttr.Cloneflags`, which
means just "pls, create new user namespace for this process", other namespace
can be specified there too. Mappings fields talk for themselves. `Size` means
size of range of mapped IDs, so you can remap many IDs without specifying each.

## PID namespaces

From `man pid_namespaces`:
```
PID namespaces isolate the process ID number space
```

That's it, your process in this namespace will have PID 1, which is sorta cool:
you're like systemd, but better. In our first part `ps awux` won't show only
our process, because we need mount namespace and remount `/proc`, but still you
can see PID 1 with `echo $$`.

## First unprivileged container

I'm pretty bad at writing big texts, so I decided to split container creation
to several parts. Today we will see only user and PID namespace creation, which
still pretty impressive. So, for adding PID namespace we need modify
`Cloneflags`:

{{< highlight go >}}
    Cloneflags: syscall.CLONE_NEWUSER | syscall.CLONE_NEWPID
{{< /highlight >}}

For this articles I created project on github: https://github.com/LK4D4/unc.
`unc` means "unprivileged container" and has nothing in common with
[runc](https://github.com/opencontainers/runc)(maybe only a little). I will tag
code for each article in repo. Tag for this article is
[user_pid](https://github.com/LK4D4/unc/tree/user_pid).
Just compile it with `go1.5` and try to run different commands from unprivileged
user in namespaces:
```
$ unc sh
$ unc whoami
$ unc sh -c "echo \$\$"
```
It doing nothing fancy, just connects your standard streams to executed process
and execute it in new namespaces with remapping current user and group to 
root user and group inside user namespace. Please read all code, there is not
much for now.

## Next steps

Most interesting part of containers is mount namespace. It allows you to have
mounts separate from host(`/proc` for example). Another interesting namespace is
network, it is little tough for unprivileged user, because you need to create
network interfaces on host first, so for this you need some superpowers from
root. In next article I hope to cover mount namespace, so it will be real container
with its own root filesystem.

Thanks for reading! I'm learning all this stuff myself right now by writing this
articles, so if you have something to say, please feel free to comment!
