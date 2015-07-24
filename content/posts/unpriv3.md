+++
title = "Unprivileged containers in Go, Part3: Mount namespace"
description = "How to execute your process with different mounts in its own rootfs"
date = "2015-07-23"
categories = ["go", "linux", "namespaces", "containers"]
+++

## Mount namespace

From `man namespaces`:

```
Mount namespaces isolate the set of filesystem mount points, meaning that
processes in different mount namespaces can have different views of the
filesystem hierarchy. The set of mounts in a mount namespace is modified using
mount(2) and umount(2).
```
So, mount namespace allows you to give your process different set of mounts. You
can have separate `/proc`, `/dev` etc. It's easy just like pass one more flag to
`SysProcAttr.Cloneflags`: `syscall.CLONE_NEWNS`. It has such weird name because
it was first introduced namespace and nobody could think that there will be more.
So, if you see `CLONE_NEWNS`, know - this is mount namespace.
Let's try to enter our container with new mount namespace. We'll see all the same
mounts as in host. That's because new mount namespace receives copy of parent
host namespace as initial mount table. In our case we're pretty restricted in
what we can do with this mounts, for example we can't unmount anything:
```
$ umount /proc
umount: /proc: not mounted
```
That's because we use "unprivileged" namespace. But we can mount new `/proc` over
old:
```
mount -t proc none /proc
```
Now you can see, that `ps` shows you only your process. So, to get rid of host
mounts and have nice clean mount table we can use pivot_root syscall to change
root from host root to some another. But first we need to write some code to
really mount something into new rootfs.

## Mounting inside root file system

So, for next steps we will need some root filesystem for tests. I will use
busybox one, because it's very small, but useful. Busybox rootfs from Docker
official image you can take
[here](https://github.com/jpetazzo/docker-busybox/raw/master/rootfs.tar). Just
unpack it to directory `busybox` somewhere:
```
$ mkdir busybox
$ cd busybox
$ wget https://github.com/jpetazzo/docker-busybox/raw/master/rootfs.tar
$ tar xvf rootfs.tar
```

Now when we have rootfs, we need to mount some stuff inside it, let's create
datastructure for describing mounts:
{{< highlight go >}}
type Mount struct {
    Source string
    Target string
    Fs     string
    Flags  int
    Data   string
}
{{< /highlight >}}
It is just arguments to `syscall.Mount` in form of structure. Now we can add some
mounts and path to rootfs(it will be just current directory for `unc`) in
addition to hostname to our `Cfg` structure:
{{< highlight go >}}
type Cfg struct {
    Path     string
    Args     []string
    Hostname string
    Mounts   []Mount
    Rootfs   string
}
{{< /highlight >}}
For start I added `/proc`(to see process tree from new PID namespaces, btw you
can't mount `/proc` without PID namespace) and `/dev`:
{{< highlight go >}}
    Mounts: []Mount{
        {
            Source: "proc",
            Target: "/proc",
            Fs:     "proc",
            Flags:  defaultMountFlags,
        },
        {
            Source: "tmpfs",
            Target: "/dev",
            Fs:     "tmpfs",
            Flags:  syscall.MS_NOSUID | syscall.MS_STRICTATIME,
            Data:   "mode=755",
        },
    },
{{< /highlight >}}

Mounting function looks very easy, we just iterate over mounts and call
`syscall.Mount`:
{{< highlight go >}}
func mount(cfg Cfg) error {
    for _, m := range cfg.Mounts {
        target := filepath.Join(cfg.Rootfs, m.Target)
        if err := syscall.Mount(m.Source, target, m.Fs, uintptr(m.Flags), m.Data); err != nil {
            return fmt.Errorf("failed to mount %s to %s: %v", m.Source, target, err)
        }
    }
    return nil
}
{{< /highlight >}}

Now we have something mounted inside our new rootfs. Time to pivot_root to it.

## Pivot root

From `man 2 pivot_root`:
```
int pivot_root(const char *new_root, const char *put_old);
...
pivot_root() moves the root filesystem of the calling process to the directory
put_old and makes new_root the new root filesystem of the calling process.

...

       The following restrictions apply to new_root and put_old:

       -  They must be directories.

       -  new_root and put_old must not be on the same filesystem as the current root.

       -  put_old must be underneath new_root, that is, adding a nonzero number
          of /.. to the string pointed to by put_old must yield the same directory as new_root.

       -  No other filesystem may be mounted on put_old.
```
So, it's taking current root, moves it to `old_root` with all mounts and makes
`new_root` as new root. `pivot_root` is more secure than `chroot`, it's pretty hard
to escape from it. Sometimes `pivot_root` isn't working(for example on Android
systems, because of special kernel loading process), then you need to use
mount to "/" with MS_MOVE flag and `chroot` there, here we won't discuss this case.

Here is the function which we will use for changing root:
{{< highlight go >}}
func pivotRoot(root string) error {
    // we need this to satisfy restriction:
    // "new_root and put_old must not be on the same filesystem as the current root"
    if err := syscall.Mount(root, root, "bind", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
        return fmt.Errorf("Mount rootfs to itself error: %v", err)
    }
    // create rootfs/.pivot_root as path for old_root
    pivotDir := filepath.Join(root, ".pivot_root")
    if err := os.Mkdir(pivotDir, 0777); err != nil {
        return err
    }
    // pivot_root to rootfs, now old_root is mounted in rootfs/.pivot_root
    // mounts from it still can be seen in `mount`
    if err := syscall.PivotRoot(root, pivotDir); err != nil {
        return fmt.Errorf("pivot_root %v", err)
    }
    // change working directory to /
    // it is recommendation from man-page
    if err := syscall.Chdir("/"); err != nil {
        return fmt.Errorf("chdir / %v", err)
    }
    // path to pivot root now changed, update
    pivotDir = filepath.Join("/", ".pivot_root")
    // umount rootfs/.pivot_root(which is now /.pivot_root) with all submounts
    // now we have only mounts that we mounted ourselves in `mount`
    if err := syscall.Unmount(pivotDir, syscall.MNT_DETACH); err != nil {
        return fmt.Errorf("unmount pivot_root dir %v", err)
    }
    // remove temporary directory
    return os.Remove(pivotDir)
}
{{< /highlight >}}
I hope that all is clear from comments, let me know if not. It is all code that
you need to have your own unprivileged container with its own rootfs. You can
try to find other rootfs among docker images sources, for example alpine linux
is pretty exciting distribution. Also you can try to mount something more inside
container.

That's all for today. Tag for this article on github is
[mnt_ns](https://github.com/LK4D4/unc/tree/mnt_ns). Remember that you should
run `unc` from unprivileged user and from directory, which contains rootfs. Here
is examples of some commands inside container(excluding logging):
```
$ unc cat /etc/os-release
NAME=Buildroot
VERSION=2015.02
ID=buildroot
VERSION_ID=2015.02
PRETTY_NAME="Buildroot 2015.02"

$ unc mount
/dev/sda3 on / type ext4 (rw,noatime,nodiratime,nobarrier,errors=remount-ro,commit=600)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,nodev,mode=755,uid=1000,gid=1000)

$ unc ps awxu
PID   USER     COMMAND
    1 root     ps awxu
```
Looks pretty "container-ish" I think :)

Previous parts:

 * [Part 1](/posts/unpriv1)
 * [Part 2](/posts/unpriv2)
