---
layout: post
title:  "C++11: Invalid Suffix on Literal Warning/Error"
date:   2021-03-06 11:21:09 +0100
---

I recently came across this unexpected error when compiling C++ code using
Clang:

```
error: invalid suffix on literal; C++11 requires a space between literal and identifier
```

I got a similar message, although as a warning, when compiling the same code
with GCC:

```
warning: invalid suffix on literal; C++11 requires a space between literal and string macro
```

The problem occurs when compiling code like this:

```c++
printf("Test %"PRIu32"\n", x);
```

Note that there is no space between the string `Test %` and `PRIu32` which is
apparently a requirement in C++11 (or newer). Fortunately, the fix is quite
simple -- just add spaces between the string and identifier. So the equivalent
code below fixes the problem:

```c++
printf("Test %" PRIu32 "\n", x);
```

But what if you have a lot of legacy code that you cannot modify? There are a
couple of flags that can disable the warnings/errors, but these are different
for Clang and GCC.

**NOTE:** An obvious way to get rid of the problem in both Clang and GCC is to
indicate a C++ standard version earlier than C++11. For example, invoking the
compiler with the flag `-std=c++03` fixes the problem, but of course, this is
not an ideal solution.

# Clang

In Clang, the `invalid suffix literal` message shows as an error by default.
You can turn it into a warning message with the
`-Wno-error=reserved-user-defined-literal` flag. Alternatively, you can
completely eliminate the warning/error message with `-Wno-reserved-user-defined-literal`.
For example,

```
clang++ -std=c++11 -Wno-error=reserved-user-defined-literal -c -o test.o test.cpp
```

**NOTE:** [This page](https://clang.llvm.org/docs/UsersManual.html#options-to-control-error-and-warning-messages)
from the Clang documentation explains how the warning and error flags
work.

# GCC

In GCC, the `invalid suffix literal` message shows as a warning by default. You
can eliminate the message with the `Wno-literal-suffix` flag. For example,

```
g++ -std=c++11 -Wno-literal-suffix -c -o test.o test.cpp
```

**NOTE:** [This page](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html)
from the GCC documentation explains how the warning and error flags work.
