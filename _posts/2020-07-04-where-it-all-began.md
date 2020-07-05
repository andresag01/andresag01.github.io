---
layout: post
title:  "'The Compiler will Optimize It' #0: Where It All Began"
date:   2020-07-04 11:21:09 +0100
---

Once upon a time, I was fresh out of university working for a company
developing a portable IoT platform for embedded devices. The idea was that
customers could use a single API to 'easily' develop edge applications in C++
for a range of supported microcontrollers.

Portable? C++? How does that work? Basically, our APIs were a collection of
abstract classes that were implemented for each microcontroller; the build
system picks the correct implementation for your configured target. We were not
in the business of developing drivers for every microcontroller under the sun.
So we normally used the vendor's Software Development Kits (SDK) to implement
our APIs.

Unfortunately, microcontrollers are often very different and so are their SDKs.
To get around this problem, we were often forced to implement layers upon
layers of glue code. But this often results in quite a bit of extra code. Also,
each call to even simple high-level API functions can turn into a few calls
before the SDK's code is actually executed.

I was soon concerned about performance, so I brought it up with my boss. He
always answered in the same way: we will be fine, **the compiler will
optimize it!** He was quite a senior engineer, so I believed this story for
the next couple of years... until I worked on a compiler.

Since then, I have found that quite a few people trust the optimization power
of the almighty compiler -- this is not unfounded. Compilers are very good at
optimizing code in *some* situations. **But compilers are no holy grail and will
often degrade your code.**

So, I have taken it upon myself to find evidence for this claim! I will blog about
the specific situations where I think the compiler caused problems as I come
across them. Hopefully, being aware of these pitfalls can help developers
better understand compilers and use them more effectively.
