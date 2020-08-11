---
layout: post
title:  "'The Compiler will Optimize It' #1: Clever Constants"
date:   2020-07-11 11:21:09 +0100
---

Sometime ago, I was testing an LLVM back-end that I had just written for an
experimental instruction set. I was trying to compile the [Mbed TLS
implementation of SHA-224 and
SHA-256](https://github.com/ARMmbed/mbedtls/blob/db09ef6d22f3043536910833c43faf425a7e0401/library/sha256.c)
when a strange-looking fragment of generated assembly caught my eye. This is
the code of interest that conditionally assigns a constant to a small buffer
depending on the selected hash algorithm (more about SHA-2
[here](https://en.wikipedia.org/wiki/SHA-2)):

```c
if( is224 == 0 )
{
    /* SHA-256 */
    ctx->state[0] = 0x6A09E667;
    ctx->state[1] = 0xBB67AE85;
    ...
    ctx->state[7] = 0x5BE0CD19;
}
else
{
    /* SHA-224 */
    ctx->state[0] = 0xC1059ED8;
    ctx->state[1] = 0x367CD507;
    ...
    ctx->state[7] = 0xBEFA4FA4;
}
```

Initially, I thought that my back-end was broken or just plain inefficient, so I
recompiled the C program using a mature LLVM back-end. Here is the code I
got when I tell LLVM to generate code for the ARM Cortex M0 processor:

```armasm
; r1 holds the is224 flag
; r0 holds the ctx pointer

  ; Compare is224 with 0
  cmp r1, #0

  ; Branch if is224 == 0 i.e. SHA-256 selected
  beq .LBB0_2
  ; Load SHA-224 constant from memory
  ldr r3, .LCPI0_1
  b .LBB0_3
.LBB0_2:
  ; Load SHA-256 constant from memory
  ldr r3, .LCPI0_0
.LBB0_3:
  ; Store constant at ctx->state[7]
  str r3, [r0, #36]

  ; Compare is224 with 0... again?
  cmp r1, #0
  ; Branch if is224 == 0 i.e SHA-256 selected
  beq .LBB0_5
  ; Load SHA-224 constant from memory
  ldr r3, .LCPI0_3
  b .LBB0_6
.LBB0_5:
  ; Load SHA-256 constant from memory
  ldr r3, .LCPI0_2
.LBB0_6:
  ; Store constant at ctx->state[6]
  str r3, [r0, #32]

  ; Compare is224 with 0... in case it changed?
  cmp r1, #0
  ...

  ; The pattern repeats for every pair of constants...

  ; Constant pool
.LCPI0_0:
  .long 1541459225              @ 0x5be0cd19
.LCPI0_1:
  .long 3204075428              @ 0xbefa4fa4
...
```

What's happening here? The compiler is repeatedly checking the `is224` flag, so
every assignment of a 32-bit constant actually costs a compare, branch, load
and store. This is actually quite dreadful assembly! It uses a lot more
instructions than it should. But why did the compiler generate this?

# Constant pools

There are a couple of reasons behind LLVM's long-winded implementation. Lets
look at the individual assignments first. The compiler generates a load of a
constant value followed by a store. This is necessary because the Thumb
instructions used by the Cortex M0 are only 16 bits; they have very short
immediates and its difficult to materialize large constants.

The compiler instead keeps track of constants, like the numbers to the right
of the assignment, using a *constant pool*. The implementation of these pools
is dependent on the compiler back-end because instruction sets have different
needs and features. For Thumb, the constants are put as close as possible to
the place where they are used. This is why you get seemingly random values
scaterred around the code when you disassemble a Thumb binary.

A key thing about constant pools is that the generated code cannot make
assumptions about the placement of constants. For example, if I have two
32-bit constants, we cannot be sure that the first is located at address *x*
and the second at *x + 1*. So the compiler cannot just generate a
`memcpy`-style loop to copy the constants from the pool to `ctx->state`,
although that would save us quite a lot of code!

# Conditionals

Lets briefly look at how LLVM operates before thinking about why it feels the
need to repeatedly test the same condition.

Compilation with LLVM is long process. First, the C code is compiled by a
front-end, like Clang, into the LLVM Intermediate Representation (IR) which is
an architecture-independent assembly-like language. The IR is then put through
an aptly called `opt` program that performs generic optimizations as instructed
by the compiler flags (e.g. -O2, -O3, -Osize etc). The `opt` output
is still an IR program that is compiled down to object code or assembly by a
back-end.

Back-ends transform the input IR into a Directed Acyclic Graph (DAG); basically
a tree-like data structure where nodes are instructions and edges are operands.
The DAG is put through several compilation passes that progressively 'lower'
the program to actual machine instructions. Finally, the object or assembly
code is emitted.

Each assignment within the if-else block in our C program is
represented by a
[SELECT](https://github.com/llvm/llvm-project/blob/898065a7b879f204874820f16e4e16ea2a961de0/llvm/include/llvm/CodeGen/ISDOpcodes.h#L597)
node in the DAG. The result of SELECT is either of two operand values depending
on a condition; think about it like C's ternary operator (e.g `cond ? trueval :
falseval`). These SELECT statements get lowered by back-ends using various
strategies. For example, the ARM back-end maps SELECTs into a [CMOV]()
pseudo-instruction which eventually gets lowered to comparison and branch
instructions for Thumb.  In other words, our simple if-else block got expanded
into 8 separete if-else blocks! Something like this:

```
if( is224 == 0 )
    ctx->state[0] = 0x6A09E667;
else
    ctx->state[0] = 0xC1059ED8;

if( is224 == 0 )
    ctx->state[1] = 0x367CD507;
else
    ctx->state[1] = 0xBB67AE85;

...
```

Thus, we get that oddly looking assembly code that could really use a rewrite.

# Other architectures

I tried compiling exactly the same program for a few other targets using LLVM:
i386, XCore and ARMv7-M (ARM Cortex M3). In all cases I got very similar
results to the code shown above, but obviously using the respective instruction
set. If you are curious, you can use [these files]({{ site.url
}}/download/compiler_optimization_1.zip) to try it out!

Something interesting also happens when you compile the program for other
architectures. If you check the code, you will note that there is a lot of
register pressure in this tiny program, so the compiler starts saving values
onto the call stack. More on this in a future post!

# Can we make it better?

I do not know a way to improve the generated code in this case short of using a
different compiler. In fact, I compiled exactly the same code using GCC, instead
of LLVM, and got much neater assembly code which uses one conditional branch
only.

# Conclusion

Know. Your. Compiler!

Compilers are not the all powerful optimizing machines that we sometimes
believe they are. Compilers can generate code that is not ideal, or just plain
inefficient.

*Did you find this interesting? Have you come across other issues with
compilers? [Get in touch](mailto:aa1399@my.bristol.ac.uk)!*
