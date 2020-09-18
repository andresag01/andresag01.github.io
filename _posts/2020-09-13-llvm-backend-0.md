---
layout: post
title:  "How to Write an LLVM Backend #0: Introduction"
date:   2020-09-13 11:21:09 +0100
---

Writing a compiler for a novel instruction set always felt like a complicated
affair to me, and in many ways, this is true. But nowadays LLVM makes the
process a lot simpler than I expected! In my humble opinion, the main
difficulty is just the lack of tutorials describing in a simple, step-by-step
fashion the process to adapt LLVM to one's needs.[^1] [^2] So this series of
blog posts is an attempt to remedy (part of) the problem by providing an
easy-to-follow tutorial to writing an LLVM backend from scratch.

In this tutorial, I will develop a backend for the basic 32-bit version of the
RISC V instruction set, ie. RV32IM. Hopefully this helps anyone who is
unfamiliar with LLVM to get started with the tool and extend it for their own
projects. There are no pre-requisites to follow this tutorial, but it helps if
you speak C++ and are familiar with [RISC
V](https://riscv.org/technical/specifications/).

In the remaining of this post, I will briefly describe LLVM's architecture and
the structure of the backend. However, I will not go into specific details here
because, if you are like me, you will forget 5 minutes (or seconds?) after you
read the long description. Details will be provided in later posts as required.

**NOTE:** You can delight yourself while reading the [LLVM User
Guides](http://llvm.org/docs/UserGuides.html) if you want the
long version.

{% include llvm-backend-contents.md %}

# LLVM Architecture

The compilation process is traditionally split in three phases. First, the
compiler's frontend takes a program as an input and transforms it into some
Intemediate Representation (IR). Second, the IR is optimized, and finally, the
compiler's backend transforms the IR into machine code. But the problem is that
traditional compilers often target one programming language and one instruction
set, so the compiler's source cannot easily be reused to, for example, emit
code for a different instruction set.

LLVM is an extremely modular implementation of the three-phase compilation
process to solve the reuse problem. The idea is that LLVM's core, i.e. the IR
and optimizer, are fixed, but the frontend and backend can be changed to
retarget the compiler for a different programming language or instruction set.
For example, we can compile C/C++ code using Clang (a frontend for LLVM) and
the x86 backend to emit code for that instruction set. We could just as easily
replace LLVM's backend with one for ARM to compile C/C++ for that instruction
set.

**NOTE:** Chris Lattner, LLVM's designer, wrote
[this](http://www.aosabook.org/en/llvm.html) interesting article about
LLVM's architecture and motivation.

**NOTE:** Each of the three phases has a dedicated executable in LLVM. `clang`
is the frontend for C/C++ (this can obviously change for different programming
languages), `opt` is the optimizer and `llc` runs the backend. Normally, we
also use clang as a driver that executes the frontend, `llc` and `opt` with the
appropriate arguments to generate IR, assembly, executable, etc.

# Code Generation at the Backend

A LLVM backend compiles IR down to object or assembly code. Each backend
targets a single architecture, but possibly multiple instruction sets. For
example, LLVM has only one backend for the ARM architecture that
can emit code for instruction sets like ARMv6 and ARMv7. Every
backend is built on top of LLVM's Target-Independent Code Generator. The code
generator is a framework that implements key algorithms like register
allocation. So broadly speaking, the task of a backend is to configure and
adapt that framework to the particular needs of its target instruction set.

The code generation has the following stages:

1. **Instruction Selection:** The input LLVM IR is mapped to instructions in
the target instruction set. At this stage, the program is using an infinite set
of virtual registers and abstract references to the function call stack.
1. **Scheduling and Formation:** Determines an ordering of the instructions.
Just to be clear, there was already an ordering to the instructions at the
instruction selection stage, but here we can choose to reorder some of those
instructions depending on the register allocation strategy or the instruction
latencies.
1. **SSA-based Machine Code Optimizations:** Performs stuff like
[peephole](https://en.wikipedia.org/wiki/Peephole_optimization) optimizations.
1. **Register Allocation:** Maps the virtual register to physical registers.
1. **Prolog/Epilog insertion:** Inserts the machine instructions at the
beginning (or *prolog*) and end (or *epilog*) of every function. These would
typically be instructions that extend the stack when entering a function or
return to the caller. Abstract stack references are also resolved since the
stack size is known at this stage.
1. **Late Machine Code Optimizations:** Probably self-explanatory.
1. **Code Emission:** Emits the object or assembly code.

So that is it for this post! Next, I will take a look at building LLVM and how
to set up a development/debugging environment...

**NOTE:** You can read more about the LLVM Target-Independent Code Generator
[here](http://llvm.org/docs/CodeGenerator.html).

# Notes

[^1]: In fairness, there are quite a few books and websites about LLVM, but most of these are general descriptions of the tool. There are also hands-on tutorials on how to write a new frontend, but not a back-end.
[^2]: [This](https://jonathan2251.github.io/lbd/) tutorial describes how to develop an LLVM backend, but I found it very hard to follow.
