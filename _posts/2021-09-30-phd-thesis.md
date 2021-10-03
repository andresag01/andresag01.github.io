---
layout: post
title: "Integrated Hardware Garbage Collection for Real-Time Embedded Systems"
date:  2021-09-30 11:21:09 +0100
---

I recently completed my thesis while pursuing a PhD in Computer Science at the
University of Bristol. The thesis, titled *Integrated Hardware Garbage
Collection for Real-Time Embedded Systems*, is shared below under the
[Creative Commons Attribution-NonCommercial 4.0
International](http://creativecommons.org/licenses/by-nc/4.0/) license.

The thesis is related to garbage collection, computer architecture, memory
management, among other topics. I continue to be interested in the subject, so
please feel free to reach out with questions, comments, ideas, thoughts, etc.
My contact details are [here]({{ site.url }}/about/) and at the bottom of this
page.

**NOTE:** A lot of the articles in this blog are about findings or ideas that I
came across while working on my PhD thesis!

# Abstract

Modern programming languages, like Python and C#, provide
productivity and trust benefits that are key in managing the growing complexity
of computer systems. However, modern language implementations rely on software
garbage collection which imposes high overheads and unpredictable pauses. This
is tolerable in large computer systems, like desktops and servers, but
impractical for real-time embedded systems. Hence modern languages are rarely
used to program embedded devices.

This thesis investigates a shift in architecture towards hardware garbage
collection to better support modern languages in embedded devices while meeting
their unique performance and real-time requirements. We present an
Integrated Hardware Garbage Collector (IHGC) that demonstrates this
approach: a collector that is tightly coupled with the processor and runs
continuously in the background. Our design allocates a memory cycle to the
collector when the processor is not using the memory. The IHGC achieves this by
careful subdivision of collection work into single-memory-access steps that are
interleaved with the processor’s memory accesses. We also introduce a static
analysis technique to guarantee that real-time programs are never paused by the
IHGC. As a result, our collector eliminates run-time overheads and is suitable
for real-time embedded systems.

The IHGC is evaluated through simulation based on a hardware implementation
model using modern fabrication technologies. Our experiments indicate that
the IHGC offers 1.5-7 times better performance compared to a conventional
processor running a software garbage collector. In addition, our static,
real-time analysis technique was evaluated through practical use cases showing
that an IHGC system meets specific timing constraints. This thesis concludes
that the IHGC delivers in real-time the benefits of garbage collected languages
without the complexity and overheads inherent in software collectors.

# Download

You can download the thesis from [here]({{ site.url }}/download/phd_thesis.pdf).

# Related Publications

The following publications are related to the contents of the thesis:

* A. Amaya García, D. May and E. Nutting, Garbage Collection for Edge
Computing, in 2020 IEEE/ACM Symposium on Edge Computing (SEC), 2020, pp.
319-319. Accessible [here](https://ieeexplore.ieee.org/document/9355637).
* A. Amaya García, D. May and E. Nutting, Integrated Hardware Garbage
Collection, ACM Transactions on Embedded Computer Systems (TECS), 2021. Accessible
[here](https://dl.acm.org/doi/10.1145/3450147).
