+++
title = "Unprivileged containers in Go, Part1: User and PID namespaces"
description = "How to create basic container in Go, first steps"
date = "2015-07-15"
categories = ["go", "linux", "namespaces", "containers"]
+++

## Unprivileged namespaces

Unprivileged(or user) namespaces are [Linux
namespaces](http://man7.org/linux/man-pages/man7/namespaces.7.html), which can
be created by an unprivileged(non-root) user. It is possible only with a usage
of user namespaces. Exhaustive info about user namespaces you can find in
manpage `man user_namespaces`. Basically for creating your namespaces you need
to create user namespace first. The kernel can take a job of creating
namespaces in the right order for you, so you can just pass a bunch of flags to
`clone` and user namespace always created first and is a `parent` for other
namespaces.

## User namespace

In user namespace you can map user and groups from host to this namespace, so
for example, your user with uid 1000 can be 0(root) in a namespace.

Mrunal Patel [introduced to
Go](https://github.com/golang/go/commit/f9d7e139552b186f4c68a3a87b470847167a9076)
support for user and groups and go 1.4.0 including it. Unfortunately, there was
security fix to linux kernel 3.18, which prevents group mappings from the
unprivileged user without disabling `setgroups` syscall. It was
[fixed](https://github.com/golang/go/commit/f5c60ff2da4851f9056120a423ce6b48624fb97e)
by me and will be released in 1.5.0 (UPD: Already released!).

For executing process in new user namespace, you need to create `*exec.Cmd`
like this:

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

Here you can see `syscall.CLONE_NEWUSER` flag in `SysProcAttr.Cloneflags`,
which means just "please, create new user namespace for this process", another
namespace can be specified there too. Mappings fields talk for themselves.
`Size` means a size of a range of mapped IDs, so you can remap many IDs without
specifying each.

## PID namespaces

From `man pid_namespaces`:
```
PID namespaces isolate the process ID number space
```

That is it, your process in this namespace has PID 1, which is sorta cool:
You are like systemd, but better. In our first part `ps awux` won't show only
our process, because we need mount namespace and remount `/proc`, but still you
can see PID 1 with `echo $$`.

## First unprivileged container

I am pretty bad at writing big texts, so I decided to split container creation
to several parts. Today we will see only user and PID namespace creation, which
still pretty impressive. So, for adding PID namespace we need to modify
`Cloneflags`:

{{< highlight go >}}
    Cloneflags: syscall.CLONE_NEWUSER | syscall.CLONE_NEWPID
{{< /highlight >}}

For this articles, I created a project on Github: https://github.com/LK4D4/unc.
`unc` means "unprivileged container" and has nothing in common with
[runc](https://github.com/opencontainers/runc)(maybe only a little). I will tag
code for each article in a repo. Tag for this article is
[user_pid](https://github.com/LK4D4/unc/tree/user_pid). Just compile it with
`go1.5` or above and try to run different commands from an unprivileged user in
namespaces:
```
$ unc sh
$ unc whoami
$ unc sh -c "echo \$\$"
```
It is doing nothing fancy, but just connects your standard streams to executed
process and execute it in new namespaces with a remapping current user and
group to root user and the group inside user namespace. Please read all code,
there is not much for now.

## Next steps

Most interesting part of containers is mount namespace. It allows you to have
mounts separate from host(`/proc` for example). Another interesting namespace
is a network, it is little tough for an unprivileged user, because you need to
create network interfaces on host first, so for this you need some superpowers
from the root. In next article, I hope to cover mount namespace - so it a real
container with own root filesystem.

Thanks for reading! I am learning all this stuff myself right now by writing
this articles, so if you have something to say, please feel free to comment!
