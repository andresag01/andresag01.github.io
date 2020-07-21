---
layout: post
title:  "The Compiler will Optimize It #2: The Power of Zero"
date:   2020-07-18 11:21:09 +0100
---

If you develop a secure application in C, you will quickly realize that
clearing unused memory is extremely important. Failing to clear memory after
use will not make your program crash, but can make an attacker very grateful!

Consider a program that decrypts a file using a key stored in a buffer. When
the decryption is complete, the buffer is freed without clearing
it. Normally, the memory allocator (the stuff implementing the venerable
`malloc`/`free` pair) does not clear it either[^1]. The memory for the
buffer is 'free', but it still contains the key that an attacker could recover.
So clearing memory is extremely important for security, and in my humbe opinion,
even more troublesome than memory management. In fact, uncleared 'resource'
issues are graded as one of the top 25 most dangerous software errors in the
[CWE](https://cwe.mitre.org/data/definitions/226.html).

But what does this have to do with compilers and optimization? Well,
the most common way to clear memory is to zero it. In C, you would do something
like this:

```c
int main(void)
{
    int ret = 0;
    char *buf;

    /* Initialization */
    buf = malloc(BLEN * sizeof(char));

    /* Do something important with buf... */
    ret = do_something(buf);

    /* Clean up  and terminate */
    memset(buf, 0, sizeof(char) * BLEN); /* Clear the buffer! */
    free(buf);

    return ret;
}
```

Compiling that code for the ARM Cortex-M0 with optimizations enabled, like
`-O2`, gets us this assembly code at the end of `main`. If you are curious,
you can use [these files]({{ url }}/download/compiler_optimization_2.zip) to
try it out!

```armasm
  ; Free temporary buffer
  mov r0, r4
  bl  free

  ; Restore return value
  mov r0, r5

  ; Pop callee-saved registers (and return)
  pop {r4, r5, r7, pc}
```

What happened? We are closing the file and freeing the buffer, but the
`memset` call to clear `buf` disappeared! This is [Dead Code
Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination) in action.
Basically, the compiler analyzed the code and decided that the zeroing of the
memory by `memset` was *dead*; it does not affect the program's correct
behavior. Therefore, the offending `memset` call can be eliminated for the
sake of performance. In other words, the compiler optimized a bit too much and
actually removed code that prevented a security vulnerability.

Let's look at a few ways to address the problem.

# The volatile keyword

A common attempt to clear the memory while conforming to the C standard is to
cast the buffer to a
[`volatile`](https://en.wikipedia.org/wiki/Volatile_(computer_programming))
type before calling memset. For example:

```
memset((volatile char *)buf, 0, BUF_SIZE * sizeof(char));
```

The `volatile` keyword is saying that the data referenced by the
pointer `buf` can change at any time, so the compiler should not optimize this
code -- at least not too aggressively. This keyword is normally used for stuff
like accessing I/O registers or multi-threading, but here its just simply a
'hack' to disable some compiler optimizations for this function call.

This simple strategy works --well, sort of-- but some compilers will still
optimize the `memset` call away as explained
[here](http://www.daemonology.net/blog/2014-09-04-how-to-zero-a-buffer.html). Why?
Basically, the compiler is clever enough to recognize that it is not really
handling a `volatile` object. In short, this does not really work...

# The C standard: memset_s

The C11 standard added an Annex K that contains stuff like bounds-checking
interfaces. Annex K has a function [`memset_s`](https://en.cppreference.com/w/c/string/byte/memset) (the `_s` might stand for
secure?) which performs mostly the same function as `memset`, but the
standard explicitly mandates that `memset_s` not be optimized away by
compliant implementations.

This sounds promising! It is what we want to securely clear memory
after use. But there is one small problem: some people hate Annex K -- with a
passion. Apparently, that annex is extremely controversial, and in my
experience, its not widely supported. Last time I checked, the C standard
libraries shipped with off-the-shelf GCC and LLVM did not support Annex K.

In summary, `memset_s` is only an acceptable solution if you are using a
compiler that supports it. But I dare say that this is not most people.

# Platform-specific functions

Platforms often have their own version of a secure `memset` that is
guaranteed to not be optimized away by the compiler -- or at least that is what
the specification says! For example,

* Visual Studio: [`SecureZeroMemory`](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/aa366877(v=vs.85))
* FreeBSD: [`explicit_bzero`](https://www.freebsd.org/cgi/man.cgi?query=explicit_bzero)[^2]
* NetBSD: [`explicit_memset`](https://netbsd.gw.com/cgi-bin/man-cgi?explicit_memset)

These APIs are ideal if your software only supports one or two systems. In this
case, I recommend checking the system's documentation to see if there is a
secure API available. But there is an obvious problem when your software is
expected to support multiple platforms as these secure APIs are not
standard. Going down this route will force you to code some tricky C
preprocessor check to work out which version of the API you are supposed to use
-- lets face it, this is not great! Also, you would need to track API changes
across several platforms as they might eventually break your code.

# Implement your own

For most compilers, it is possible to implement a secure version of `memset`
that will not get optimized away. Of course, this is not guaranteed to work
across the board, but you can get pretty close.

A good technique to prevent the compiler from optimizing your secure `memset`
is to hide your implementation behind the linker. Normally, C source
files are compiled into object code independently of each other. The object
code files are then stitched together by the linker at the end of the
compilation process. But the linker rarely perform any agressive optimizations
on the input object code, so its unlikely that your secure `memset`
implementation will be optimized away.

If you are looking for inspiration, take a look at the [Mbed TLS
implementation](https://github.com/ARMmbed/mbedtls/blob/8f4f9a8dafc3ecf0c36d4b6588a8d9edf05720e5/library/platform_util.c#L69)
of a secure `memset`.

# Conclusion

Writing secure software is hard, specially when the compiler works against you!

There does not seem to be a reliable, standard and broadly compatible solution
to clear a memory buffer. Personally, even if I settle on a solution I would
check the compiler's machine code output to ensure that the buffers with
sensitive information are correctly clear. This task can be tedious, but you
can automate some of it using a debugger like GDB. There is an example of how
to do this [here](https://github.com/ARMmbed/mbedtls/blob/c7da1fe3812ce0d3ee0e351e67118d46af540519/tests/scripts/test_zeroize.gdb).

# Notes

[^1]: More generally, an 'uncleared' resource is anything that may contain sensitive information and is not cleared when it is no longer needed. This includes, dynamically allocated memory, files, memory allocated on the stack, etc.
[^2]: There is also some GNU documentation about `explicit_bzero` [here](https://www.gnu.org/software/libc/manual/html_node/Erasing-Sensitive-Data.html).
