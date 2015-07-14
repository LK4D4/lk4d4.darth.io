+++
title = "Developing Arduino with Docker"
description = "Developing Arduino with docker"
date = "2015-07-13"
categories = ["arduino", "docker"]
+++

I'm using Gentoo and using Arduino on Gentoo isn't very easy: 
[Arduino on Gentoo Linux] (http://playground.arduino.cc/Linux/Gentoo).

It is easy with Docker though. Let's see how we can upload our first program
to Arduino Uno without installing anything apart from Docker.

### Kernel Modules

For Arduino Uno I need to enable 
```
Device Drivers -> USB support -> USB Modem (CDC ACM) support
```
as module.

Then I compiling and loading it with
```
make modules && make modules_install && modprobe cdc-acm
```
in my `/usr/src/linux`. At last I connect Arduino and see it as `/dev/ttyACM0`.

### Installing ino

For this we just need image from `hub.docker.com`:
```
docker pull coopermaa/ino
```
It's slightly outdated, but I sent
[PR](https://github.com/coopermaa/docker-ino/pull/1) to use new base image,
because that's how we do this in opensource world. Anyway it works great. Let's
create script for calling ino through Docker, add next script to your $PATH
```
#!/bin/sh
docker run --rm --privileged --device=/dev/ttyACM0 -v $(pwd):/app coopermaa/ino $@
```
and call it ino. Don't forget
```
chmod +x ino
```

Alternatively you can use alias in `.bashrc`:
```
alias ino='docker run --privileged \
  --rm \
  --device=/dev/ttyACM0 \
  -v $(pwd):/app \
  coopermaa/ino'
```
but script worked better with my vim.

### Uploading program

Let's create program from template and upload it to board:
```
$ mkdir blink && cd blink
$ ino init -t blink
$ ino build && ino upload
```
Whoa! It's alive!

### Vim integration

I'm using [Vim plugin for ino](https://github.com/jplaut/vim-arduino-ino/), you
can easily install it with any plugin manager for vim. You don't need anything
special, it'll just work. You can compile and upload your sketch with
`<Leader>ad`.

### Known issues

For using `ino serial` you need to add `-t` to `docker run` arguments to your
script. It works pretty weird though, you need to kill process
`/usr/bin/python /usr/local/bin/ino serial` by hands every time, but it works
and looks not so bad.

Also files created by `ino init` will belong to root, which isn't very
convenient.

### That's all!

Thank you for reading and special thanks to
[coopermaa](https://github.com/coopermaa) for ino image.
