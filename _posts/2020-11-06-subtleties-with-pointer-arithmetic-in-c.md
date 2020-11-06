---
layout: post
title:  "Subtleties with Pointer Arithmetic in C"
date:   2020-11-06 11:21:09 +0100
---

I came across an interesting bug while porting
[MicroPython](https://github.com/micropython/micropython) to run on a
simulator for an experimental computer architecture. Executing the following C
code raised errors:

```c
while (--d >= dig) {
  // Some other code goes here...
}
```

The variable `dig` is a pointer to the start of an array. `d` is another
pointer to the same array (but not necessarily the same array index) such that
`d == &dig[n]` where `n >= 0`. The expectation in this code is that the loop
will terminate when `d` points to one element *before* the start of the array.
But this behaviour is actually undefined according to the C standard, and in
fact, it caused an exception in my experimental computer architecture.

I dutifully reported the
[issue](https://github.com/micropython/micropython/issues/6066) to the
developers. But the issue encouraged a discussion on what is (not) allowed
when dealing with pointers in C. So I decided
to take a closer look at the [C89](https://www.pdf-archive.com/2014/10/02/ansi-iso-9899-1990-1/ansi-iso-9899-1990-1.pdf), [C99](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf) and [C11](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) standards and put together a list
of (what looks to me like) subtleties when using pointers.

# Object Bounds

The C standard, since C89, is surprisingly clear about what are valid pointers.
In fact, the wording has not changed when comparing the three standards. The
relevant text is in Section 6.3.6 from the C89 standard (and 6.5.6.8 in the
other standards) states:

> if the expression P points to the i-th element of an array object, the
> expressions (P)+N ... and (P)-N ... point to, respectively, the i+n-th and
> i-n-th elements of the array object, provided they exist. Moreover, if the
> expression P points to the last element of an array object, the expression
> (P)+1 points to one past the last element of the array object ... If both
> the pointer operand and the result point to elements of the same array
> object, or one past the last element of the array object, the evaluation
> shall not produce an overflow; otherwise, the behavior is undefined...

In summary

* A pointer to anywhere within an object is valid.
* A pointer to one past the end of an object is valid.
* Any other pointer, like the bug I described at the beginning of the post, is **undefined**.

Note that undefined is not a good thing at all! It just means "good luck",
anything can happen depending on the compiler, architecture, etc, which is
obviously not ideal when writing stable, reliable programs.

# Relational Operators

C allows the relational operators (<, >, <=, >=) to compare pointers depending
on the relative locations in the address space of the objects pointed to. But
there are some rules. The wording has actually changed a little since the C89
standard, so I will quote 6.5.8.5 from C11 (the equivalent in C89 is in
Section 6.3.4).

> When two pointers are compared... If two pointers to object types both
> point to the same object, or both point one past the last element of the same
> array object, they compare equal...

No surpises so far and I think the text is fairly clear. The paragraph
continues like this

> If the objects pointed to are the members of the same aggregate object,
> pointers to the structure members are declared later compare greater than
> pointers to members declared earlier in the structure, and pointers in array
> elements with larger subscript values compare greater than pointers to
> elements of the same array with lower subscript values...

Again, no surprises. This is basically saying that `&p[0] < &p[1]` and that
for for a struct `s` with members `a` and `b` we have `&s.a < &s.b` if
`struct { char a; char b } s`. Lets continue

> All pointers to members of the same union object compare equal. If the
> expression P points to the element of an array object and the expression Q
> points to the last element of the same array object, the pointer expression
> Q+1 compares greater than P...

Once more, no surprises. And here comes the interesting bit

> In all other cases, the behavior is undefined.

This is very curious because it looks to me that comparing, for example,
pointers to two *different* objects is undefined.

# NULL pointer

The C89 standard does define, in Section 6.2.2.3 (or 6.3.2.3.3 in other
standards), a null pointer as

> an integral constant expression with the value 0, or such an expression cast
> to type void *

Also, the `NULL` macro is defined in `stddef.h` as a null pointer constant. So
there should be nothing stopping a programmer from redefining NULL to another
constant value, say -1, but that would not be compliant.

# Casts

In C, it is possible to cast pointer and integer types, but the standard is
quite vague about what actually happens. First, lets take a look at casts from
integer to pointer. Here is what the C11 standard says in 6.3.2.3.5:

> An integer may be converted to any pointer type... the result is
> implementation-defined, might not be correctly aligned, might not point to an
> entity of the referenced type, and might be a trap representation.

Also, this is that 6.3.2.3.6 says about pointer to integer casting:

> Any pointer type may be converted to an integer type... the result is
> implementation-defined. If the result cannot be represented in the integer
> type, the behavior is undefined...

In other words, use with *extreme* caution and only when absolutely necessary.
For example, integer to pointer casting is unavoidable in drivers if the
machine uses memory-mapped I/O.

Note that the C standard hints why the casts might not succeed. Pointers could
be of larger size than the integer type used in the cast e.g. casting a 64-bit
pointer to a 32-bit integer. Also, not all computer architectures implement
pointers as plain memory addresses with straight-forward integer
representations. Pointers could be abstract references, using
[handles](https://en.wikipedia.org/wiki/Handle_(computing)), which do not map
neatly to integer values or the implementation does not want to allow it; think
garbage collection or exotic computer architectures (specially older ones).
