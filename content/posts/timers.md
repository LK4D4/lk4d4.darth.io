---
title: ""
description: ""
date: "2016-10-21"
draft = true
categories:
    - "go"
---

# How Do They Do It: Timers in Go

This article covers internal implementation of timers in Go. Note that there are
a lot of links to Go repo in this article, I recommend to follow them to understand
material better.

## Timers

Timers in Go just do something after a period of time. The user interface for
timers located in the standard package [time](https://golang.org/pkg/time/).
In particular, timers are [time.Timer](https://golang.org/pkg/time/#Timer),
[time.Ticker](https://golang.org/pkg/time/#Ticker) and less obvious timer
[time.Sleep](https://golang.org/pkg/time/#Sleep).
It's not clear from documentation how timers work exactly. Some people thing that
each timer spawns it's own goroutine which exists until timer is spawned, because
that's how we'd implement timers in "naive" way in Go. We can check that assumption
with small program:
```go
package main

import (
	"fmt"
	"os"
	"runtime/debug"
	"time"
)

func main() {
	debug.SetTraceback("system")
	if len(os.Args) == 1 {
		panic("before timers")
	}
	for i := 0; i < 10000; i++ {
		time.AfterFunc(time.Duration(5*time.Second), func() {
			fmt.Println("Hello!")
		})
	}
	panic("after timers")
}
```
It prints all goroutines traces before timers spawned if run without arguments
and after timers spawned if any argument is passed. We need those shady panics
because otherwise there is no easy way to see runtime goroutines - they're excluded
from `runtime.NumGoroutines`. Let's see how many goroutines Go spawns in case of
before spawning any timers:
```
go run afterfunc.go 2>&1 | grep "^goroutine" | wc -l
4
```
and after spawning 10k timers:
```
go run afterfunc.go after 2>&1 | grep "^goroutine" | wc -l
5
```
Whoa! It's only one goroutine, in my case its trace looks like:
```
goroutine 5 [syscall]:
runtime.notetsleepg(0x5014b8, 0x12a043838, 0x0)
        /home/moroz/go/src/runtime/lock_futex.go:205 +0x42 fp=0xc42002bf40 sp=0xc42002bf10
runtime.timerproc()
        /home/moroz/go/src/runtime/time.go:209 +0x2ec fp=0xc42002bfc0 sp=0xc42002bf40
runtime.goexit()
        /home/moroz/go/src/runtime/asm_amd64.s:2160 +0x1 fp=0xc42002bfc8 sp=0xc42002bfc0
created by runtime.addtimerLocked
        /home/moroz/go/src/runtime/time.go:116 +0xed
```
Let's see closer why is it so exactly.

## runtime.timer
All timers are based on the same data structure -
[runtime.timer](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L15).
In order to add new timer, you need to instantiate `runtime.timer` and pass it
to function
[runtime.startTimer](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L64).
Here is example from `time` package:
```go
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime,
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}
```
So, here we're converting duration to exact timestamp `when` timer should call
function `f` with argument `c`. There are three types of function `f` used in
package time:

* [sendTime](https://github.com/golang/go/blob/release-branch.go1.7/src/time/sleep.go#L116) -
sends current time to channel or discards it if send blocks. Used in
[time.Timer](https://github.com/golang/go/blob/release-branch.go1.7/src/time/sleep.go#L80)
and
[time.Ticker](https://github.com/golang/go/blob/release-branch.go1.7/src/time/tick.go#L34).

* [goFunc](https://github.com/golang/go/blob/release-branch.go1.7/src/time/sleep.go#L153) -
executes some function in goroutine. Used in
`[time.AfterFunc](https://github.com/golang/go/blob/release-branch.go1.7/src/time/sleep.go#L145).

* [goroutineReady](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L81) -
wakes up specific goroutime. Used in
[runtime.timeSleep](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L48)
which is linked to `time.Sleep`.

So, now we understand how timers look like in runtime and what they supposed to
do. Now let's see how runtime stores timers them and call functions when it's time to
call them.

## runtime.timers

[runtime.timers](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L28)
is just a [Heap data structure](https://en.wikipedia.org/wiki/Heap_(data_structure)).
Heap is very useful when you want to repeatedly find extremum (minimum or maximum) among
some elements. In our case extremum is a timer with closest `when` to the current
time. Very convenient, isn't it?
[siftupTimer](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L238)
and [siftdownTimer](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L255)
functions  are used for maintaining heap properties.
But data structures don't work on their own; something should use them. In our 
case it's just one goroutine with function
[timerproc](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L154).
It's spawned on
[first timer start](https://github.com/golang/go/blob/release-branch.go1.7/src/runtime/time.go#L114).

## runtime.timeproc

It's kinda hard to describe what's going on without source code, so this section
will be in form of commented Go code. Code is direct copy from `src/runtime/time.go`
file with added comments.
```
// Add a timer to the heap and start or kick the timerproc if the new timer is
// earlier than any of the others.
func addtimerLocked(t *timer) {
	// when must never be negative; otherwise timerproc will overflow
	// during its delta calculation and never expire other runtime·timers.
	if t.when < 0 {
		t.when = 1<<63 - 1
	}
	t.i = len(timers.t)
	timers.t = append(timers.t, t)
	// maintain heap invariant
	siftupTimer(t.i)
	// new time is on top
	if t.i == 0 {
		// siftup moved to top: new earliest deadline.
		if timers.sleeping {
			// wake up sleeping goroutine, put to sleep with notetsleepg in timerproc()
			timers.sleeping = false
			notewakeup(&timers.waitnote)
		}
		if timers.rescheduling {
			// run parked goroutine, put to sleep with goparkunlock in timeproc()
			timers.rescheduling = false
			goready(timers.gp, 0)
		}
	}
	if !timers.created {
		// run timerproc() goroutine only once
		timers.created = true
		go timerproc()
	}
}

// Timerproc runs the time-driven events.
// It sleeps until the next event in the timers heap.
// If addtimer inserts a new earlier event, addtimerLocked wakes timerproc early.
func timerproc() {
	// set timer goroutine
	timers.gp = getg()
	// forever loop
	for {
		lock(&timers.lock)
		// mark goroutine not sleeping
		timers.sleeping = false
		now := nanotime()
		delta := int64(-1)
		// iterate over timers in heap starting from [0]
		for {
			// there is no more timers, exit iterating loop
			if len(timers.t) == 0 {
				delta = -1
				break
			}
			t := timers.t[0]
			delta = t.when - now
			if delta > 0 {
				break
			}
			// t.period means that it's ticker, so change when and move down
			// in heap to execute it again after t.period.
			if t.period > 0 {
				// leave in heap but adjust next time to fire
				t.when += t.period * (1 + -delta/t.period)
				siftdownTimer(0)
			} else {
				// remove from heap
				// this is just removing from heap operation:
				// - swap first(extremum) with last
				// - set last to nil
				// - maintain heap: move first to its true place with siftdownTimer.
				last := len(timers.t) - 1
				if last > 0 {
					timers.t[0] = timers.t[last]
					timers.t[0].i = 0
				}
				timers.t[last] = nil
				timers.t = timers.t[:last]
				if last > 0 {
					siftdownTimer(0)
				}
				// set i to -1, so concurrent deltimer won't do anything to
				// heap.
				t.i = -1 // mark as removed
			}
			f := t.f
			arg := t.arg
			seq := t.seq
			unlock(&timers.lock)
			if raceenabled {
				raceacquire(unsafe.Pointer(t))
			}
			// call timer function without lock
			f(arg, seq)
			lock(&timers.lock)
		}
		// if delta < 0 - timers is empty, set "rescheduling" and park timers
		// goroutine. It will sleep here until "goready" call in addtimerLocked.
		if delta < 0 || faketime > 0 {
			// No timers left - put goroutine to sleep.
			timers.rescheduling = true
			goparkunlock(&timers.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
			continue
		}
		// At least one timer pending. Sleep until then.
		// If we have some timers in heap, we're sleeping until it's time to
		// spawn soonest of them. notetsleepg will sleep for `delta` period or
		// until notewakeup in addtimerLocked.
		// notetsleepg fills timers.waitnote structure and put goroutine to sleep for some time.
		// timers.waitnote can be used to wakeup this goroutine with notewakeup.
		timers.sleeping = true
		noteclear(&timers.waitnote)
		unlock(&timers.lock)
		notetsleepg(&timers.waitnote, delta)
	}
}
```
There are two variables which I think deserve explanation: `rescheduling` and
`sleeping`. They both indicate that goroutine was put to sleep, but different
synchronization mechanisms are used, let's discuss them.

- `sleeping` is set when all "current" timers are processed, but there is more
which we need to spawn in future. It uses OS-based synchronization, so it calls
some OS syscalls to put to sleep and wake up goroutine and syscalls means it spawns
OS threads for this.
It uses [note](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/runtime2.go#L131)
structure and next functions for synchronization:
	* [noteclear](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/lock_futex.go#L125) -
	resets note state, just sets key to 0.
	* [notetsleepg](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/lock_futex.go#L199) - 
	puts goroutine to sleep until `notewakeup` is called or after some period of time (in case of timers it's time until next timer).
	* [notewakeup](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/lock_futex.go#L129) - 
	wakes up goroutine which called `notetsleepg`.
`notewakeup` might be called in `addtimerLocked` if new timer is "earlier" than
previous "earliest" timer.

- `rescheduling` is set when there is no timers in our heap, so nothing to do.
It uses go scheduler to put goroutine to sleep with function
[goparkunlock](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/proc.go#L264).
Unlike `notetsleepg` it does not consume any OS resources, but also does not
support "wakeup timeout", so it can't be used instead of `notetsleepg` in
`sleeping` branch.
[goready](https://github.com/golang/go/blob/2f6557233c5a5c311547144c34b4045640ff9f71/src/runtime/proc.go#L264)
function is used for waking up goroutine when new timer added with `addTimerLocked`.

## Conclusion

We learned how Go timers work "under the hood" - runtime neither uses one goroutine per
timer, nor timers are "free" to use. It's important to understand how things work
to avoid premature optimizations. Also, we learned that it's quite easy to read
runtime code and you shouldn't be afraid to so.
I hope you enjoyed this reading and will share info to your fellow Gophers.
