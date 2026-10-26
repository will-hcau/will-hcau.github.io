---
title: "Why Old Binaries Still Work: Understanding ABI and Backward Compatibility"
date: 2026-10-21 13:00:00 +0900
categories: [Software Engineering]
tags: [c-language, abi, backward-compatibility, shared-libraries, linux]
description: "Explore dynamic linking and binary compatibility through small C experiments, disassembly, and a struct-layout bug caught by AddressSanitizer."
---

**Have you ever wondered why an executable written for Windows 95 might still run on Windows 11—without being recompiled?**

Not every old program works unchanged, and operating-system compatibility involves more than shared libraries. But the question leads to a fascinating software-engineering challenge: **how can new implementations continue to support binaries built decades earlier?**

That longevity does not happen by accident. **Microsoft and operating-system developers put substantial engineering effort into preserving backward compatibility** while their platforms evolve. Maintaining existing interfaces is part of that work; compatibility layers and targeted fixes can also help older applications keep running. Raymond Chen describes this practical effort in Microsoft's [The Old New Thing: application compatibility layers](https://devblogs.microsoft.com/oldnewthing/20100311-00/?p=14643).

We routinely update shared libraries without rebuilding every application that uses them. An executable built against an older library may continue to run with a newer implementation—even though its machine code has not changed.

How is that possible?

The answer is a contract called the **Application Binary Interface**, or **ABI**.

Before diving into how to design a library that can evolve without breaking existing applications, we first need to understand **what an ABI is** and **the basic rules for preserving it**. That is the focus of this first post; techniques for evolving an ABI will follow in the next one.

In this post, we will explore that contract through small C experiments. We will inspect how an executable calls a shared library, replace the library without recompiling the application, and deliberately break compatibility in two different ways.

Our experiments use **Linux x86-64**, **GCC**, and **ELF shared libraries**. Other platforms have different mechanisms, but the central idea remains the same:

> **An old executable does not need the original implementation. It needs the contracts it was built to depend on.**

## API versus ABI

An **Application Programming Interface (API)** describes the source-level interface programmers use:

```c
int add(int a, int b);
```

The **ABI** describes the binary-level rules that make the compiled caller and implementation work together:

- How functions are identified.
- Where arguments are passed.
- Where return values are placed.
- How data types and structures are represented in memory.
- Which registers and runtime conventions each side must respect.

Two pieces of source code can appear compatible while their compiled binaries disagree about these rules.

Likewise, two binaries can link successfully but disagree about what an operation means. That is a separate concern: **behavioral compatibility**.

## 1. Build a tiny shared library

We begin with three files.

### calc.h

```c
#ifndef CALC_H
#define CALC_H

int add(int a, int b);

#endif
```

### calc.c

```c
#include "calc.h"

int add(int a, int b)
{
    return a + b;
}
```

### app.c

```c
#include <stdio.h>
#include "calc.h"

int main(void)
{
    printf("add(2, 3) = %d\n", add(2, 3));
    return 0;
}
```

Build the library and application in separate stages:

```sh
gcc -Wall -Wextra -g -O0 -fPIC -c calc.c -o calc.o
gcc -shared -o libcalc.so calc.o

gcc -Wall -Wextra -g -O0 -c app.c -o app.o
gcc -o app app.o -L. -lcalc -Wl,-rpath,'$ORIGIN'
```

Here:

- **`-fPIC`** generates position-independent code for the shared library.
- **`-shared`** produces a shared library.
- **`-L.`** adds the current directory to the build-time library search path.
- **`-lcalc`** selects `libcalc.so` in this build.
- **`$ORIGIN`** lets the runtime loader search beside the executable.

The single quotes preserve `$ORIGIN` for the loader rather than allowing the shell to expand it. The Linux loader documents this token and its search rules in [`ld.so(8)`](https://man7.org/linux/man-pages/man8/ld.so.8.html).

Run the application:

```sh
./app
```

```text
add(2, 3) = 5
```

The important detail: **the implementation of `add()` was not copied into `app`**. It remains in the shared library.

## 2. Follow the call in disassembly

Inspect the executable:

```sh
objdump -d -Mintel app
```

In our build, the relevant part of `main()` was:

```asm
1171: mov esi,0x3
1176: mov edi,0x2
117b: call 1060 <add@plt>
1180: mov esi,eax
```

For this function under the Linux x86-64 System V calling convention:

| Register | Purpose |
|---|---|
| `edi` | First integer argument: `2` |
| `esi` | Second integer argument: `3` |
| `eax` | Integer return value |

The caller already contains those assumptions. The library must agree with them.

But the call goes to **`add@plt`**, not directly to the library implementation.

### PLT and GOT: an indirect route

Our executable contains this stub:

```asm
0000000000001060 <add@plt>:
1060: endbr64
1064: jmp QWORD PTR [rip+0x2f5e]  # 3fc8 <add@Base>
```

The **Procedure Linkage Table (PLT)** provides the call stub. That stub jumps through a pointer slot in the **Global Offset Table (GOT)**.

Conceptually:

```text
main()
  |
  | call
  v
add@plt
  |
  | jump through a GOT pointer
  v
add() inside libcalc.so
```

This indirection allows the library's actual address to be determined at runtime.

The exact assembly and addresses can vary with compiler, linker, and security settings.

## 3. How does the loader find add()?

Disassembly shows instructions, but the executable also contains **dynamic-linking metadata**.

Inspect its dependencies:

```sh
readelf -d app
```

Relevant entries from our build:

```text
(NEEDED)  Shared library: [libcalc.so]
(NEEDED)  Shared library: [libc.so.6]
(RUNPATH) Library runpath: [$ORIGIN]
```

The executable requests `libcalc.so`; it does not embed an absolute address for its `add()` implementation.

Next, inspect the imported symbol:

```sh
readelf --dyn-syms app | grep add
```

```text
3: 0000000000000000  0 FUNC GLOBAL DEFAULT UND add
```

**`UND`** means that `add` is undefined in the executable and must be supplied elsewhere.

The relocation identifies the pointer slot to update:

```sh
readelf -rW app | grep add
```

```text
0000000000003fc8 ... R_X86_64_JUMP_SLOT ... add + 0
```

Finally, inspect the library's exported symbol:

```sh
nm -D --defined-only libcalc.so
```

```text
00000000000010f9 T add
```

For this ordinary function in our shared library, the runtime address is:

```text
runtime address of add = library load bias + 0x10f9
```

The value `0x10f9` is a **symbol virtual-address value**, not necessarily a file offset. ELF symbol and relocation structures are described in [`elf(5)`](https://man7.org/linux/man-pages/man5/elf.5.html).

The dynamic linker searches the applicable symbol scope, finds `add`, and places its resolved address in the GOT slot. In our normal run, `libcalc.so` supplies it.

### When is the pointer resolved?

Our executable also contained:

```text
(FLAGS)   BIND_NOW
(FLAGS_1) Flags: NOW PIE
```

Therefore, its function relocations are resolved during startup. Other configurations can use **lazy binding**, resolving a function on its first call.

Crucially, **`app` never needed to know `0x10f9`**. A replacement library can move the function elsewhere, and runtime symbol resolution can still locate it.

## 4. Replace the implementation, keep the contract

Create `calc_ver2.h` with the same declaration:

```c
#ifndef CALC_VER2_H
#define CALC_VER2_H

int add(int a, int b);

#endif
```

Then create `calc_ver2.c`:

```c
#include <stdio.h>
#include "calc_ver2.h"

int add(int a, int b)
{
    puts("Version 2: add()");
    return a + b;
}
```

**Keep the original executable.** `app` was compiled using the original `calc.h` and linked against the original `libcalc.so`, whose API is `int add(int a, int b)`. It is **not** rebuilt against Version 2's header or library.

Build only the new library:

```sh
gcc -Wall -Wextra -g -O0 -fPIC -shared \
    calc_ver2.c -o libcalc_ver2.so
```

The executable still requests the filename **`libcalc.so`**. To test a replacement without overwriting the original:

```sh
mkdir -p v2
cp libcalc_ver2.so v2/libcalc.so

LD_LIBRARY_PATH="$PWD/v2" ./app
```

Output:

```text
Version 2: add()
add(2, 3) = 5
```

For this ordinary, non-secure execution, `LD_LIBRARY_PATH` takes priority over our executable's `DT_RUNPATH`. See the loader's [library search rules](https://man7.org/linux/man-pages/man8/ld.so.8.html).

**We changed the implementation without rebuilding the application.**

We also verified that the executable's SHA-256 remained identical before and after the experiment. You can check this yourself with `sha256sum app`.

An alternative is:

```sh
LD_PRELOAD=./libcalc_ver2.so ./app
```

However, this demonstrates **symbol interposition**, not a complete dependency replacement: the original `libcalc.so` is still required. The directory-based test more directly models replacing the library.

## 5. Break the function signature

Now create `calc_ver3.h` with a changed interface:

```c
#ifndef CALC_VER3_H
#define CALC_VER3_H

int add(int a, int b, int c);

#endif
```

Our `calc_ver3.c` implementation is:

```c
#include <stdio.h>
#include "calc_ver3.h"

int add(int a, int b, int c)
{
    printf("Version 3: add(a=%d, b=%d, c=%d)\n", a, b, c);
    return a + b + c;
}
```

**Again, do not recompile or relink `app`.** We are still using the executable built against the **original two-argument header and original `libcalc.so`**, not against Version 2 or Version 3. Only the library selected at runtime changes.

Build and install it into another test directory:

```sh
gcc -Wall -Wextra -g -O0 -fPIC -shared \
    calc_ver3.c -o libcalc_ver3.so

mkdir -p v3
cp libcalc_ver3.so v3/libcalc.so

LD_LIBRARY_PATH="$PWD/v3" ./app
```

Two actual runs produced:

```text
Version 3: add(a=2, b=3, c=207566776)
add(2, 3) = 207566781
```

```text
Version 3: add(a=2, b=3, c=798467480)
add(2, 3) = 798467485
```

Why?

The old executable still supplies only two arguments:

```asm
mov esi,0x3
mov edi,0x2
call add@plt
```

The new implementation also reads **`edx`**, where it expects the third integer argument. But the caller never prepared that register for `c`.

**A missing argument is not automatically zero.** The library interprets whatever bits happen to be present. This is **undefined behavior**, not a dependable way to obtain an arbitrary number.

### Why did symbol resolution succeed?

Because the ordinary C symbol is still named **`add`**. The dynamic linker does not validate its parameter count and types against the header used to compile the application.

```text
Symbol resolution: succeeds
Calling contract:  broken
```

### Key Point 1: Preserve exported function signatures

> **Existing binaries depend on argument types, argument count, return types, and calling conventions.**

To add functionality, retain the old entry point and introduce a new one:

```c
int add(int a, int b);
int add3(int a, int b, int c);
```

## 6. Break a struct without changing the function declaration

Function signatures are not the whole contract.

For a separate experiment in a `struct_demo` directory, our original `calc.h` defines:

```c
#ifndef CALC_H
#define CALC_H

struct Values {
    int a;
    int b;
};

int add(const struct Values *values);

#endif
```

The original `calc.c` sums the two fields:

```c
#include "calc.h"

int add(const struct Values *values)
{
    return values->a + values->b;
}
```

Our new `app.c` allocates and initializes an object using the old header:

```c
#include <stdio.h>
#include <stdlib.h>
#include "calc.h"

int main(void)
{
    struct Values *values = malloc(sizeof(*values));
    if (values == NULL)
        return 1;

    values->a = 2;
    values->b = 3;

    printf("App: sizeof(Values)=%zu\n", sizeof(*values));
    fflush(stdout);
    printf("add(values) = %d\n", add(values));

    free(values);
    return 0;
}
```

On our platform, this object occupies **8 bytes**.

Now the replacement library's `calc_ver2.h` uses a larger definition:

```c
#ifndef CALC_VER2_H
#define CALC_VER2_H

struct Values {
    int a;
    int b;
    int c;
};

int add(const struct Values *values);

#endif
```

Its function declaration still looks the same. But `calc_ver2.c` reads all three members:

```c
#include <stdio.h>
#include "calc_ver2.h"

int add(const struct Values *values)
{
    printf("Library v2: sizeof(Values)=%zu, a=%d, b=%d, c=%d\n",
           sizeof(*values), values->a, values->b, values->c);
    return values->a + values->b + values->c;
}
```

Build the two libraries and one executable from within `struct_demo`:

```sh
gcc -Wall -Wextra -g -O0 -fPIC -shared calc.c -o libcalc.so
gcc -Wall -Wextra -g -O0 app.c -L. -lcalc \
    -Wl,-rpath,'$ORIGIN' -o app

mkdir -p v2
gcc -Wall -Wextra -g -O0 -fPIC -shared \
    calc_ver2.c -o v2/libcalc.so
```

The old application and new library disagree:

```text
Byte offset     0–3       4–7       8–11
              +---------+---------+---------+
Library sees: |    a    |    b    |    c    |
              +---------+---------+---------+
App owns:     <----- 8 bytes ----->
                                  ^
                           Outside the object
```

### The misleading successful run

Running `./app` with the original library produced:

```text
App: sizeof(Values)=8
add(values) = 5
```

Running `LD_LIBRARY_PATH="$PWD/v2" ./app` produced:

```text
App: sizeof(Values)=8
Library v2: sizeof(Values)=12, a=2, b=3, c=0
add(values) = 5
```

It appears correct—but it is not safe.

In this run, the invalid read happened to observe zero. Nothing in the interface guarantees that result.

The replacement library's disassembly exposes the problem:

```sh
objdump -d -Mintel --disassemble=add v2/libcalc.so
```

One relevant instruction is:

```asm
mov ecx,DWORD PTR [rax+0x8]
```

With `rax` holding the struct pointer, this reads four bytes starting at offset **8**, immediately beyond the old object.

### AddressSanitizer reveals the bug

Build separate instrumented versions using **`-fsanitize=address`**:

```sh
mkdir -p asan/v2

gcc -Wall -Wextra -g -O0 -fsanitize=address -fPIC -shared \
    calc.c -o asan/libcalc.so

gcc -Wall -Wextra -g -O0 -fsanitize=address \
    app.c -Lasan -lcalc -Wl,-rpath,'$ORIGIN' -o asan/app

gcc -Wall -Wextra -g -O0 -fsanitize=address -fPIC -shared \
    calc_ver2.c -o asan/v2/libcalc.so

LD_LIBRARY_PATH="$PWD/asan" ./asan/app
LD_LIBRARY_PATH="$PWD/asan/v2" ./asan/app
```

Both the executable and library code are instrumented, and the final executable is linked with the sanitizer enabled. The **same instrumented executable** works with the original library, but fails with the replacement:

```text
ERROR: AddressSanitizer: heap-buffer-overflow
READ of size 4
...
0 bytes after 8-byte region
```

**AddressSanitizer** detects out-of-bounds accesses and other memory errors by adding runtime instrumentation. Its [official documentation](https://clang.llvm.org/docs/AddressSanitizer.html) explains the build requirements and diagnostic reports.

This is more than a wrong answer: it is a **memory-safety bug**. An out-of-bounds read can potentially expose data or crash a process; a corresponding write can corrupt adjacent memory. The security impact depends on how the API is used.

### Key Point 2: Public struct layouts are part of the ABI

> **Existing binaries depend on structure sizes, alignment, and member offsets—not just function declarations.**

A pointer carries an address, not the size or version of the object at that address.

Changing a struct passed **by value** can be even more disruptive: its size and contents may affect whether arguments are passed in registers or memory.

Not every size change breaks every usage. Ownership, allocation, and which fields are accessed all matter. But our experiment shows why appending a field is **not automatically safe** when old applications allocate the object.

Most importantly:

> **“It returned the expected answer” is not proof of compatibility.**

## 7. Preserve meaning, not just machine-level rules

Suppose we keep:

```c
int add(int a, int b);
```

but change the implementation to:

```c
return a - b;
```

Arguments still arrive correctly. The return value uses the right register. Symbol resolution succeeds.

Yet the application's expectations are broken.

### Key Point 3: Preserve the API's behavioral contract

**Binary compatibility is necessary, but it is not sufficient for backward compatibility.**

The behavioral contract can include:

- What an operation computes.
- Return values and error codes.
- Ownership and object lifetime.
- Side effects.
- Thread-safety guarantees.
- Blocking versus non-blocking behavior.

This does not mean every implementation detail or accidental bug must remain forever. It means distinguishing internal changes from changes to behavior callers were promised they could rely on.

Our Version 2 diagnostic print is itself an observable side effect. It is useful for this experiment, but adding output would not be behaviorally compatible with an API that promised silence.

## 8. Can GCC and Clang work together?

**Yes, when they target a compatible ABI and runtime environment.**

The compiler brand is not the contract. For our ordinary Linux x86-64 C interface, a GCC-built application can normally call a Clang-built shared library using the same platform ABI.

GCC's documentation explains that interoperability requires a shared ABI and warns that compiler options can alter it. It covers calling conventions, data representation, and object formats. [GCC: Binary Compatibility](https://gcc.gnu.org/onlinedocs/gcc/Compatibility.html)

Clang documents GCC compatibility as an intentional design goal, though not an unconditional guarantee for all extensions and configurations. [Clang Compiler User's Manual](https://clang.llvm.org/docs/UsersManual.html)

Important conditions include compatible architecture, public declarations, packing, enum representation, calling conventions, and runtime dependencies.

Also, do not confuse **native object-code compatibility** with interchangeability of compiler-specific **LTO intermediate representations**.

> **GCC and Clang can interoperate because they follow a shared target ABI—not simply because both compile C.**

## Conclusion: backward compatibility is a promise

Dynamic linking lets a library's implementation—and the locations of its functions—change without rewriting the application.

But the loader cannot establish that the replacement still honors every assumption compiled into that application.

Our experiments revealed three essential rules:

1. **Preserve exported function signatures.**
2. **Treat public struct layouts as part of the ABI.**
3. **Preserve the API's behavioral contract.**

Backward compatibility is not guaranteed by a familiar filename, a matching symbol name, or a successful test run. It depends on preserving the contracts existing applications rely on.

That leaves the next question: **how can a library grow without freezing its interface forever?**

In the next post, we will explore techniques such as **opaque handles**, **extensible structures**, **new entry points**, and **symbol versioning**—ways to add features without abandoning old binaries.
