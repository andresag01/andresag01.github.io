---
layout: post
title:  "How to Write an LLVM Backend #2: Setting Up a New Backend"
date:   2020-09-20 11:21:09 +0100
---

Developing an LLVM backend is not a particularly glamorous affair. You will
soon realize that it is largely an exercise in copy-pasting and adapting code
from other existing backends. In fact, LLVM developers in online forums
suggest getting started by "copying an existing backend, rename it and modify
it to suit your needs". Sounds simple, except that even relatively small
backends, like Lanai or XCore, are rather complex and the code is
not easy to follow!

I will take a slightly different approach in this series of posts. We will be
using an existing LLVM backend as a starting point, *but* I have stripped out
most of the code and reduced it to the bare minimum needed to compile a
(tiny) program. The stripped-down backend, called RISCW, is simple enough to
help understand the LLVM Target-Independent Code Generator without getting
bogged down in the details. In the remaining of this post I will use the RISCW
backend to show how to set up a new LLVM backend. We will also see how to
build LLVM with an experimental backend and even compile a (very simple) C program down to
assembly.

**NOTE:** The code for the RISCW backend can be found
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750).

{% include llvm-backend-contents.md %}

# LLVM Triple and ELF Configuration

We start by configuring a new target triple for our backend. For historical
reasons, the triple is a way of encoding information about the target such as
the architecture, vendor and operating system. Here are the steps to configure
a new triple:

1. Declare a new architecture for the triple in
`llvm/include/llvm/ADT/Triple.h` (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-185a32f321f9849c118053f8133d402a6f97ba3bf1a1f3ee14064d7716537c45)).
1. Provide a type conversions between string and Triple architecture (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R57),
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R150),
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R293)
and
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R427)).
1. Indicate what type of object format the backend generates, e.g. ELF, COFF,
etc. RISCW will work with ELF only (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R702)).
1. Indicate the architecture variant, e.g. 32- or 64-bit, and the pointer size (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R1349),
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R1394)
and
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-a6431c77551c3d00b1ac20938a6a95bd3f50c5ed7d126e7f25b5b55fe55767c7R1265)).

**NOTE:** You can find more information about triples
[here](https://wiki.osdev.org/Target_Triplet) and
[here](https://github.com/andresag01/llvm-project/blob/274cfea0f9662f0ed49f6132b0424323d0b11750/llvm/include/llvm/ADT/Triple.h#L22).


**NOTE:** The architecture variant does not necessarily imply a pointer size.
For example, it is not always the case that pointers are 64-bit when compiling
for RV64. The pointer size is usually given by the ABI which could be `ilp32`
(i.e. `int`, `long` and pointers are 32 bits) in a 64-bit machine.

Since RISCW uses ELF, this is a good time to configure the following parameters
related to that:

1. Create a new machine architecture enum for RISCW (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-154cb05b335ba56c78d9e24e3a0020741fb990f5eeb6972c39f8d7eb46d03215R314)).
This integer is encoded in the `e_machine` field of the ELF header. The
value is not arbitrary; it must match the registered architecture types for
the ELF format e.g. 0xF3 for RISCV.  But we will set it to an unused value
for now.
1. Declare the ELF relocation types (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-154cb05b335ba56c78d9e24e3a0020741fb990f5eeb6972c39f8d7eb46d03215R636)
and
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-cb4cf96df700591fda3ce01a86a303c5402af582993fc7e385aeb65b19c129ab)).
Again, these are architecture-dependent and those for RISCV are listed
[here](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md#relocations).
At this stage, we will simply put place-holders for RISCW.
1. The file format name (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-233e5d37dd601982330d1168360551576a27b0d09e39cb23ab0b4c020898c444R1083)).
1. Indicate the target triple for a given class (see
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-233e5d37dd601982330d1168360551576a27b0d09e39cb23ab0b4c020898c444R1166)).
Currently, the class in the ELF header is a byte that encodes whether the
format is 32- or 64- bit.

**NOTE:** Take a look at
[wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) for
more information on the ELF file format.

# Driver Configuration

Recall that we are using clang to compile the input C code down to LLVM IR. But
clang is not just our frontend compiler, it is also a driver, like GCC, that
*drives* the compilation pipeline to transform an input C program into another
representation e.g. C to assembly or object code. Therefore, we need to modify
clang to tell it

1. that there is a new RISCW backend target with a particular feature set. For
example, clang needs to be aware whether RISCW is 32- or 64-bit.
1. what is the RISCW compilation pipeline. For instance, what assembler should
it use? what linker? which include paths? etc

We can tell clang about RISCW by adding a new target class `RISCWTargetInfo`
that is instanciated alongside the existing LLVM targets as shown
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-c8988599cfd2b4c841116e418af4bf906062bd837504abb472c51328fe6d4202).
The class is declared and defined
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-65d0aabd511c54bbbe83b2e8cd966a8a33343d0d9975c20fbee6f1107c556cf8)
and
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-6b8a32b344f2f5cb34138f64efb5257fcf22c4016f7f209e373f1a168298e629).
There are a few important things to highlight in this code:

* `RISCWTargetInfo` describes the data layout via a string. This
string encodes information like the bits in a pointer and stack alignment
requirements.
* The target may indicate what is the size of basic C data types.
* A function `RISCWTargetInfo::getTargetDefines()` indicates what C
preprocessor macros are defined at compile-time. For example,
[these](https://github.com/riscv/riscv-toolchain-conventions#cc-preprocessor-definitions)
macros are defined when compiling code using the RISCV target. The macros
generally describe what architecture is used, the ABI, any enabled/disabled
architectural features, etc.

**NOTE:** A backend might target multiple instruction sets, ABIs, etc, so the
driver configuration must be changed according to the selected target triple.
For example, the `RISCVTargetInfo` changes the data layout string depending on
whether the triple contains `riscv32` or `riscv64`.

**NOTE:** Take a look
[here](https://github.com/andresag01/llvm-project/blob/llvm-backend-riscw/clang/include/clang/Basic/TargetInfo.h)
at the declaration of the parent class `TargetInfo` of `RISCWTargetInfo`. It
contains a lot more options that you can configure.

Configuring the toolchain is relatively straight-forward. We simply need to
implement a `RISCWToolChain` class that inherits from `Toolchain` as shown
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-319fa9a953c803f15928a00ac01bc1f0ab138ced2eea28d475efc5b5ce502688)
and
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-accd85034ba352dbdfea64b3dccba92aa6a8ca9de804b2e73151e7bab169874b).
The code is mostly self-explanatory, but there are a lot more options that your
target can modify by overriding the members of the `ToolChain` class (see
[here](https://github.com/andresag01/llvm-project/blob/llvm-backend-riscw/clang/include/clang/Driver/ToolChain.h)).

# Creating a New Target

Each backend has a separate directory under `llvm/lib/Target` where the
majority of its code is contained. We will not go into the details in this post
(we will do that later on) because even a small backend, like RISCW, has a lot
of files. For now, it suffices to say that we can broadly classify the files
into three groups:

* **TableGen files:** The LLVM Target-Independent Code Generation framework
implements an elaborate pattern matching algorithm to select instructions for
the input program. The patterns used for matching are described to LLVM using
the TableGen syntax. Additionally, TableGen files also describe important
architecture-specific features like the number of registers and the procedure
calling convention.
* **Build files:** The directory for every backend must be declared
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750#diff-bbc88c8ee4f01236c43353c60fc06e961a341d9d93d78a144b0416146a2070e9R34),
otherwise it will not be built. Additionally, the top directory for our target,
i.e. `llvm/lib/Target/RISCW`, and every subdirectory must contain two build
files: `CMakeLists.txt` and `LLVMBuild.txt`. The former adds source files and
any subdirectories to the build target while the latter sets simple build
parameters for the target component. Parameters include the library name,
required libraries for linking, etc.
* **C++ classes:** The C++ files comprise the bulk of the backend code and
implement everything from simple configuration options to more complex
instruction selection functionality that is not (or cannot) be captured by
TableGen.

# Building the Experimental Backend

Now that everything is set up, we can build LLVM with our new RISCW backend.
But we cannot simply modify the `-DLLVM_TARGETS_TO_BUILD` option to the CMake
command from the previous post to include RISCW because that backend is still
experimental. Instead, we use the `-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD` option
like this:

```sh
cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD="ARM;Lanai;RISCV" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="RISCW" -DCMAKE_BUILD_TYPE="Debug" -DLLVM_ENABLE_ASSERTIONS=On ../llvm
ninja
```

When the build is complete, you can check that RISCW is now an available target
as follows:

```sh
$ ./build/bin/llc --version
LLVM (http://llvm.org/):
  LLVM version 10.0.1
  DEBUG build with assertions.
  Default target: x86_64-unknown-linux-gnu
  Host CPU: znver2

  Registered Targets:
    arm     - ARM
    armeb   - ARM (big endian)
    lanai   - Lanai
    riscv32 - 32-bit RISC-V
    riscv64 - 64-bit RISC-V
    riscw   - 32-bit RISC-V         <== YAY!!
    thumb   - Thumb
    thumbeb - Thumb (big endian)
```

# Compiling our First C Program

Our RISCW backend can only emit two instructions `add` and `ret`, but it cannot
properly handle function calls, stacks and pretty much everything else! So
we will restrain ourselves and only compile this tiny function:

```c
int test(int a, int b)
{
    return a + b;
}
```

And voilà! We get this code:

```armasm
	.text
	.file	"test.c"
	.globl	test                    ; -- Begin function test
	.type	test,@function
test:                                   ; @test
; %bb.0:                                ; %entry
	add	x0, x1, x0
	ret
.Lfunc_end0:
	.size	test, .Lfunc_end0-test
                                        ; -- End function
	.ident	"clang version 10.0.1 (https://github.com/llvm/llvm-project 89f2d2cc3bba7cb12cee346b3205cb0335e758cd)"
	.section	".note.GNU-stack","",@progbits
```

Again, there are a lot of things missing *and* the code is actually incorrect
because `x0` in RISCV is a read-only register hard-wired to 0. But I think we
achieved our objective: we set up a minimal LLVM backend that we can easily
extend with more features.

**NOTE:** Make sure you set clang's `-target riscw` and llc's `-march=riscw` if
you are using the commands from the previous post to compile the `test`
function above.

**NOTE:** Attempting to compile more complex programs will result in a
`cannot select...` error. Give it a try if you are interested.

**NOTE:** You can instruct the compiler to print debug information by passing
the `-debug` option to `llc`.
