---
layout: post
title:  "How to Write an LLVM Backend #1: Getting Started"
date:   2020-09-13 11:21:09 +0100
---

I normally set up my environment and experiment with what is already out there
before writing code for a new project. So that is exactly what I am going to
do here. In this post, I will show how to download and compile LLVM and other
tools that will be useful for debugging. We will also see how to compile,
assemble, link and run programs using existing LLVM backends and the GNU
toolchain.

{% include llvm-backend-contents.md %}

# Environment

I am using Ubuntu, but you should be able to replicate the steps in other
systems with (relatively) few changes. You will need the following tools to
build the software.

* Makefile
* C/C++ Compiler -- I am using GCC 9.2.1
* autotools
* CMake
* Ninja
* Git
* A lot of patience...

**NOTE:** I might have forgotten something here, but the build system should
kindly tell you via an error ;).

# Compiling LLVM

The LLVM maintainers have set up [this](https://github.com/llvm/llvm-project)
convenient repo that contains LLVM along with other parts of the
toolchain such as Clang. So go ahead and clone that repo.

```sh
git clone https://github.com/llvm/llvm-project
```

We will be using LLVM 10.0.1 in this series of posts, so I recommend you
build that version of the software. Bear in mind that LLVM changes very
quickly, so some of the code shown here will *not* work in older/newer
versions. However, the principles should roughly be the same.

LLVM uses CMake to generate the build files for the build system. There are a
few possible target build systems: Ninja, Makefiles, Visual Studio and XCode. I
normally use Ninja as I feel that its faster in my system (although I have no
evidence to back up that statement!). You can change the build system by
modifying the `-G` argument to the `cmake` command below.

The CMake files have a lot of options that I encourage you to explore as some
can be very helpful for debugging. You can delight yourself reading about all
the build options [here](https://llvm.org/docs/CMake.html). For now, I will use
the following:

1. `-DLLVM_ENABLE_PROJECTS` to build Clang alongside the rest of the compiler
1. `-DLLVM_TARGETS_TO_BUILD` to specify a list of backends to build. Looking at
the output of other backends is enlightening and helpful for debugging, but the
build will take ages if you add too many.
1. `-DCMAKE_BUILD_TYPE` to ask for a `Debug` build.
1. `-DLLVM_ENABLE_ASSERTIONS=On` to enable assertions. Again, helpful for
debugging.

Anyways, here is how you build LLVM after cloning the repo.

```sh
cd llvm-project
git checkout llvmorg-10.0.1
mkdir build
cd build
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD="ARM;Lanai;RISCV" -DCMAKE_BUILD_TYPE="Debug" -DLLVM_ENABLE_ASSERTIONS=On ../llvm
ninja
```

**NOTE:** You can find more information about building LLVM
[here](https://llvm.org/docs/GettingStarted.html) and
[here](https://llvm.org/docs/CMake.html).

**NOTE:** You can pass the `-j <NUM_JOBS>` option for Ninja to indicate how
many jobs you want to run in parallel. A very high `<NUM_JOBS>` causes the
build to crash in my system with a `collect2: ld...` error message.

# Compiling the GNU Toolchain for RISC V

You are probably a bit confused about why I am suggesting to build GCC for
RISC V. Aren't we writing our own compiler backend anyways?

We are building GCC because, at least initially, we want to use GCC's assembler
and linker to test the code generated by our LLVM backend. Recall that there
are a lot of stages for the compilation process. At the early stages of our
development we will have the following structure:

* Clang to compile C code to LLVM IR
* LLVM to optimize the IR
* Our LLVM backend to compile the IR down to assembly
* GCC to assembly and link the executable

Use the following commands to download, build and install GCC for RISC V.

```sh
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
mkdir build
cd build
../configure --with-arch=rv32gc --with-abi=ilp32
make
make install
```

**NOTE:** Make sure you build the GCC toolchain for the right variant of the
instructionset, i.e. RV32, as the build system's default is RV64!

**NOTE:** The GNU toolchain supports multiple ABIs for RISC V, like ilp32,
ilp32d and ilp32f, depending on whether you want soft floating-point, hard
floating-point, etc.

# Compile a simple C program

Everything its now set up to build and run our first program, although not with
our own backend (yet!). Lets start with a simple C program:

```c
#include <stdio.h>

int main(void)
{
    printf("Hello world!\n");

    return 0;
}
```

First, compile the C code to LLVM IR using Clang. Our program is using the
standard library function `printf` from the `stdio.h` header, so the compiler
will throw errors if it cannot find that header file. I will be using the
standard C library that is packaged with GCC for RISC V, so I had to use the
`-isystem` argument. This adds an include path with the location of the much
needed header files to the list of search paths that Clang's preprocessor is
using.

```sh
clang -O2 -emit-llvm -target riscv64 -isystem <PATH_TO_GCC>/riscv64-unknown-elf/include -c test.c -o test.bc
```

The previous command created a `test.bc` file with the LLVM IR, but thats not
very human-readable. We can disassemble that file using the following command:

```sh
llvm-dis test.bc
```

Now lets compile the IR down to assembly using the backend that is packaged
with out LLVM download using the command:

```sh
llc -march=riscv64 -O2 -filetype=asm test.bc -o test.S
```

Generating the program's binary is fairly straight-forward with GCC. I split it
into two steps, but you can use a single command if you prefer.

```sh
riscv64-unknown-elf-gcc -c test.S -o test.o
riscv64-unknown-elf-gcc test.o -o test
```

Finally, we can run the program using a simulator or real hardware.