---
layout: post
title:  "Dynamic Linking to a Different libc"
date:   2021-05-08 11:21:09 +0100
---

Sometimes it is necessary to run a dynamically linked program using an
alternative libc from the system's. In the remaining of this post, we will take
a look at a few ways of doing that in Linux.

**NOTE:** The contents of this post were put together from several
StackOverflow answers that were helpful to me when dealing with this problem
(see [here](https://stackoverflow.com/questions/847179/multiple-glibc-libraries-on-a-single-host)
and [here](https://unix.stackexchange.com/questions/120015/how-to-find-out-the-dynamic-libraries-executables-loads-when-run)).

# Finding Dynamic Library dependencies

Lets start from the beginning: How do you find the dynamic library dependencies
of an existing executable? There are two easy ways to do this:

1. `ldd`: This command is perhaps the most straight-forward method. Here is an
example.
```bash
$ ldd $(which ls)
        linux-vdso.so.1 (0x00007ffe867fa000)
        libcap.so.2 => /usr/lib/libcap.so.2 (0x00007f6faa5bd000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f6faa3f0000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f6faa607000)
```
The output is mostly self-explanatory: The first line is Linux's [vDSO (Virtual
Dynamic Shared Object)](https://man7.org/linux/man-pages/man7/vdso.7.html)
which is a small shared object that we do not have to care about. libcap and
libc in the second and third lines of output are library dependencies while the
last line is the [dynamic linker/loader](https://linux.die.net/man/8/ld-linux)
(this runs implicitly whenever you execute a dynamically linked program).
1. `readelf`: This is the classic utility to display information about ELF
files (see [here](https://linux.die.net/man/1/readelf)). Run it like this to
display the contents of the ELF's dynamic section:
```bash
$ readelf -d $(which ls)
Dynamic section at offset 0x21a58 contains 28 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x4000
```
Again, the content is mostly self-explanatory.

**NOTE:** These commands can be useful to track down which libraries you are
missing!

**NOTE:** `ldd` and `readelf` do not show information about libraries loaded at
run-time using [`dlopen`](https://man7.org/linux/man-pages/man3/dlopen.3.html).
Use other utilities for that!

**NOTE:** The advantage of `readelf` over `ldd` is that the it shows you the
ELF's [rpath](https://en.wikipedia.org/wiki/Rpath) (if any). Read on for more
information about the rpath.

# Linking Against a Specific libc

By default, ELF files are linked against the system's libc (usually found at
`/usr/lib/`). We will explore three ways to link against another libc.

**NOTE:** The libc consists of many shared objects. The versions of these files
must match since they contain some hard-coded information (e.g. file paths), so
do not attempt to link against shared objects from different libcs.

## Method 1: `LD_LIBRARY_PATH`

This is perhaps the easiest way if you have an already built ELF executable.
The `LD_LIBRARY_PATH` is an environment variable with a colon-separated list of
paths telling the linker *where* to look for libraries. Here is a usage
example:

```bash
$ export LD_LIBRARY_PATH=/home/andres/Repos/glibc-2.33/install/lib
$ ldd $(which ls)
        linux-vdso.so.1 (0x00007fff889ec000)
        libcap.so.2 => /usr/lib/libcap.so.2 (0x00007f61144fa000)
        libc.so.6 => /home/andres/Repos/glibc-2.33/install/lib/libc.so.6 (0x00007f6114337000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f6114544000)
```

Notice that `ldd` resolved the `libc.so.6` to the alternative libc indicated in
the `LD_LIBRARY_PATH`. However, my alternative libc does not contain
`libcap.so.2`, so this shared object was resolved using the system's libraries
at `/usr/lib`.

Changing the `LD_LIBRARY_PATH` works well in most cases, but it cannot
overwrite the [run-time search path
(rpath)](https://en.wikipedia.org/wiki/Rpath) hard-coded into an ELF
executable. We will look at one way to change the rpath later in this post.

## Method 2: Using the Dynamic Linker/Loader

The [dynamic linker/loader](https://linux.die.net/man/8/ld-linux) is run
implicitly when you run a dynamically linked executable. It looks like a shared
object file, but can be run explicitly to configure dynamic linking options.
Perhaps its most useful command line option is `--library-path` which allows
you to indicate a colon-separated list of locations to resolve the dynamic
libraries. Here is an example of running `ls` with an alternative libc that I
downloaded and compiled from source:

```
/home/andres/Repos/glibc-2.33/install/lib/ld-linux-x86-64.so.2        \
    --library-path /home/andres/Repos/glibc-2.33/install/lib:/usr/lib \
    $(which ls)
```

**NOTE:** Take a look at the dynamic linker/loader command line options by
running it with `--help`. For example, `/lib64/ld-linux-x86-64.so.2 --help` in
my system.

**NOTE:** The dynamic linker/loader is a little bit temperamental, so only pass
absolute paths to it. Avoid using relative paths like `..`.

## Method 3: Set the rpath at Link-Time

The idea is to set the ELF's rpath at link-time by passing compiler flags. This
is obviously inconvenient if you already have an executable and do not want (or
cannot) link it once again, but is still an effective mechanism.

`--rpath` and `--dynamic-linker` are the linker options to configure the rpath
and the dynamic linker/loader. Here is a usage example:

```bash
cc -o test -lm                                                                          \
    -Wl,--rpath=/home/andres/Repos/glibc-2.33/install/lib                               \
    -Wl,--dynamic-linker=/home/andres/Repos/glibc-2.33/install/lib/ld-linux-x86-64.so.2 \
    test.c
```

The `readelf` and `ldd` outputs confirm that the rpath for my `test` executable
is set to the desired location, so the dynamic linker/loader will use the
alternative shared objects for libc and libm.

```bash
$ readelf -d test

Dynamic section at offset 0x2dd8 contains 28 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000f (RPATH)              Library rpath: [/home/andres/Repos/glibc-2.33/install/lib]
...

$ ldd test
        linux-vdso.so.1 (0x00007ffcfea8f000)
        libm.so.6 => /home/andres/Repos/glibc-2.33/install/lib/libm.so.6 (0x00007f7564e67000)
        libc.so.6 => /home/andres/Repos/glibc-2.33/install/lib/libc.so.6 (0x00007f7564ca4000)
        /home/andres/Repos/glibc-2.33/install/lib/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f7564faf000)

```

**NOTE:** According to [this](https://stackoverflow.com/a/44710599/2111578)
post, it is also possible to change the rpath of an existing ELF with
[patchelf](https://github.com/NixOS/patchelf), but I have never tried this. The
advange over Method 3 is that patchelf does not require you to link the program
again.
