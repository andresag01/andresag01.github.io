---
layout: post
title:  "How to Write an LLVM Backend #3: Configuring the Build System"
date:   2020-11-17 11:21:09 +0100
---

Previously, we mentioned that each LLVM backend has a separate directory with
most of its code in `llvm/lib/Target`.  Also, LLVM relies on CMake to generate
the build files for an actual build system like Make, Ninja, etc. In this post,
we will take a close look at some of the CMake build files.

**NOTE:** The code for the RISCW backend can be found
[here](https://github.com/andresag01/llvm-project/commit/274cfea0f9662f0ed49f6132b0424323d0b11750).

{% include llvm-backend-contents.md %}

# Build Files in the Target Backend Directory

Recall that the RISCW backend has its own directory at `llvm/lib/Target/RISCW`.
That directory, along with any subdirectory, has two build files, i.e.
`CMakeLists.txt` and  `LLVMBuild.txt`, that describe the structure of the
backend module, provide linking information, etc. The RISCW backend is
currently very simple, so the directory structure looks like this:

```
llvm/lib/Target/RISCW
|- CMakeLists.txt
|- LLVMBuild.txt
|- Other C++, TableGen, etc files
|- TargetInfo/
|  |- CMakeLists.txt
|  |- LLVMBuild.txt
|  |- Other C++, TableGen, etc files
|- MCTargetDesc/
   |- CMakeLists.txt
   |- LLVMBuild.txt
   |- Other C++, TableGen, etc files
```

The `CMakeLists.txt` is nothing particularly special -- every build system
relying on CMake would have them.[^1] These file simply contain CMake commands
to drive the generation of the build files. In LLVM backends the commands do
things like indicating the C++ source files to compile or generating TableGen
execution commands. The `LLVMBuild.txt` files are descriptions of the
components in the current directory.

**The `LLVMBuild.txt`**

Lets look at the contents of `llvm/lib/Target/RISCW/LLVMBuild.txt`.

```
[common]
subdirectories = MCTargetDesc TargetInfo

[component_0]
type = TargetGroup
name = RISCW
parent = Target
has_asmprinter = 1

[component_1]
type = Library
name = RISCWCodeGen
parent = RISCW
required_libraries = AsmPrinter CodeGen Core MC RISCWDesc RISCWInfo
  SelectionDAG Support Target

add_to_library_groups = RISCW
```

This file tells the build system that the RISCW backend has two components.
There is a top-level target called `RISCW` of type `TargetGroup`. This type
is used by backend and the build system actually has some special treatment for
them. Also, `Target` is the parent component of `RISCW`. Note that this naming
matches the directory structure of LLVM backends i.e. `llvm/lib/Target/RISCW`.
The name `RISCW` name is not arbitrary, it must match the definitions in
the backend's TableGen files.

LLVM backends can offer a range of optional features like assembly printing,
assembly parsing, etc. The `LLVMBuild.txt` file indicates which features are
supported. For example, the last line tells the build system that the `RISCW`
backend only implements assembly printing.

**NOTE:** Take a look at the code from existing backends, like ARM or RISCV, to
see what other features they enable besides `has_asmprinter`.

The `LLVMBuild.txt` file also defines a second component called `RISCWCodeGen`
of type `Library` whose parent is `RISCW`. Also, `RISCWCodeGen` indicates
what libraries it requires, e.g. AsmPrinter, CodeGen, etc; missing libraries
give rise to linking errors when building LLVM.

Each subdirectory of `llvm/lib/Target/RISCW` also contains a `LLVMBuild.txt`
file defining its own component with `RISCW` as a parent.

**The `CMakeLists.txt`**

Lets consider the contents of `llvm/lib/Target/RISCW/CMakeLists.txt`.

```cmake
set(LLVM_TARGET_DEFINITIONS RISCW.td)

tablegen(LLVM RISCWGenRegisterInfo.inc -gen-register-info)
tablegen(LLVM RISCWGenInstrInfo.inc -gen-instr-info)
# Other TableGen commands

add_public_tablegen_target(RISCWCommonTableGen)

# RISCWCodeGen should match with LLVMBuild.txt RISCWCodeGen
add_llvm_target(RISCWCodeGen
  RISCWAsmPrinter.cpp
  # Other files
)

# Should match with "subdirectories =  MCTargetDesc TargetInfo" in LLVMBuild.txt
add_subdirectory(TargetInfo)
add_subdirectory(MCTargetDesc)
```

The `set` command at the top of the file defines `LLVM_TARGET_DEFINITIONS` to
`RISCW.td`. This file normally contains a few top-level definitions and pulls
in other TableGen definitions by including other files -- we will look more
carefully at TableGen in later posts, but feel free to take a look at
`RISCW.td` in the repository.

We then see a few `tablegen` commands in the CMake file. These tell the build
system that it should create the `RISCWGen*.inc` files from `*.td` files in
the backend using the TableGen utility. These `RISCWGen*.inc` files are actually
conventional C++ code and you can find them in the build directory at
`build/lib/Target/RISCW` after compiling LLVM; they can sometimes be useful for
debugging if a little hard to read.

The next command, i.e. `add_llvm_target`, specifies the C++ files to build in
the current directory, but *not* its subdirectories. The first argument to the
`add_llvm_target` command is the name of the target and should match the
component name defined in `LLVMBuild.txt`.

Lastly, the `CMakeLists.txt` contains commands indicating the subdirectories
that CMake should look at. In the RISCW case, there are only two: `TargetInfo`
and `MCTargetDesc`. The `CMakeLists.txt` in the subdirectories are similar, but
a lot simpler!

**NOTE:** The naming convention of the files in the backend is often important
and should obviously match the contents of the build files. For example, the
`set(LLVM_TARGET_DEFINITIONS RISCW.td)` CMake command requires that `RISCW.td`
exists! Make sure you double-check these because the build errors can be a
little cryptic.

# Notes

[^1]: You can find more information about CMake [here](https://cmake.org/overview/).
