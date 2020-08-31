---
layout: post
title:  "'The Compiler will Optimize It' #3: Lazy isn't so Lazy"
date:   2020-08-27 11:21:09 +0100
---

[Lazy evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation) is one of
those features that has been adopted by many programming languages. The idea is
that expressions are only evaluated when their value is *needed*.
It turns out that the result of some expressions is not needed at all even
though the program's execute path appears to suggest so, thus [lazy
evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation) can reduce a
program's run-time. A classic example of this feature in action is the
following simple expression in C:

```c
A && B
```

The idea is that expression `A` is evaluated first. Then, expression `B` is
only evaluated if the result of `A` is 1 (or true) because otherwise the whole
expression evaluates to 0 (or false) anyways. In fact, this is written in the
semantic description of the logical AND operator in [C
standard](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1548.pdf):[^1]

>If the first operand compares equal to 0, the second operand is not evaluated.

Interestingly, the compiler does not always take advantage of lazy evaluation
to optimize the code as you might expect. For example, consider the following
C code:

```c
if (y < 32 && x > (CONSTANT << y))
{
    return (CONSTANT >> y);
}
else
{
    return CONSTANT;
}
```

I compiled that with LLVM for the
[XCore](https://www.xmos.ai/download/The-XMOS-XS1-Architecture(1.0).pdf) and
got the following. If you are curious, you can use [these
files](/download/compiler_optimization_3.zip) to try it out!

```armasm
  ; Evaluate y < 32
  ldc r2, 32
  lsu r3, r1, r2

  ; Load CONSTANT into a register
  ldw r2, cp[.LCPI0_0]

  ; Evaluate x > (CONSTANT << y)
  shl r11, r2, r1
  lsu r0, r11, r0

  ; Use the expression results
  bt r0, .LBB0_1
  ...
```

So... what happened there? Basically, the compiler evaluated *both* expressions
before making a decision. However, it would have probably been more efficient
to evaluate the logical AND lazily.

A consequence of this is that the compiler did not respect the C standard. In
fact, other violations of the standard often happen, such as when performing
pointer arithmetic. But the compiler gets away with it because there is no
obvious observable mistake in the code from the user's point of view. Of
course, the situation would have been very different if the instructions to
evaluate `x > (CONSTANT << y)` had side effects when errors occur. For
example, an architecture could raise an exception when bitwise-shifting left
by a value greater than the word size e.g. `1 << 64` in a 32-bit machine. For
this reason, LLVM backend has a [hasSideEffects
flag](https://github.com/llvm/llvm-project/blob/b16ac94419b73b2979b3d81855d283bf58cfd6f7/llvm/include/llvm/Target/Target.td#L565) to prevent the compiler from
reordering the code in ways that could lead to undesirable, observable
behavior.

# Conclusion

Compilers do not always evaluate code lazily. More importantly, compilers
do not necessarily respect the programming language's standard -- at least not
in observable ways. So watch out for situations where this could be a problem,
and if you ever write a compiler, make sure you prevent reorderings of the code
that lead to observable incorrect behavior.

# Notes

[^1]: See Section 6.5.13 in the standard.
