---
title: "Why C++ ABI Compatibility Is Hard"
date: 2026-11-19 00:00:00 +0900
categories: [Software Engineering]
tags: [cpp, abi, backward-compatibility, shared-libraries, gcc, clang, msvc]
description: "Explore C++ ABI compatibility through name mangling, cross-compiler experiments, a private-member layout change, and the limits of PImpl."
---

Why would a library use C++ internally but expose a C-compatible interface to applications?

Two familiar platforms illustrate this separation between the programming model and the binary boundary:

- **CUDA's Driver API** exposes functions such as `cuInit()` and represents objects through handles such as `CUcontext` and `CUstream`. NVIDIA describes it as a handle-based API in its [Driver API documentation](https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html).
- **Classic Win32** exposes functions that operate on handles such as `HWND`. Applications manipulate windows without knowing the internal object layout. Microsoft explicitly explains that these objects are [not C++ classes exposed to the application](https://learn.microsoft.com/en-us/windows/win32/learnwin32/what-is-a-window-).

These examples show an interface design, not proof that every implementation behind those interfaces is written in C++. Nor are ABI concerns the only reason to expose a C-compatible interface; language interoperability matters too.

But they raise an interesting question: **what additional contracts do we expose when the binary interface itself consists of C++ functions and classes?**

In the [first ABI post]({% post_url 2026-10-21-understanding-abi-and-backward-compatibility %}), we explored the difference between source compatibility and binary compatibility using C. This time, we will look at C++ through a few small experiments.

We will focus on two separate questions:

1. Can an application and library built by different compilers agree on the interface?
2. Can an old application continue using a library after its class implementation changes?

The answers are related, but they are not the same problem.

## What does a C++ function look like in a binary?

Start with a function:

```cpp
int add(int a, int b);
```

Our first experiments used GCC 13.3 on Linux x86-64. The library exported this symbol:

```text
_Z3addii
```

This is **name mangling**: the compiler encodes information about the function into its binary name.

For this example:

| Part | Meaning |
|---|---|
| `_Z` | Mangling prefix |
| `3add` | Three-character function name: `add` |
| `i` | First parameter: `int` |
| `i` | Second parameter: `int` |

Now add an overload:

```cpp
int add(int a, int b);
int add(double a, double b);
```

The library exports two distinct symbols:

```text
_Z3addii    -> add(int, int)
_Z3adddd    -> add(double, double)
```

The compiler resolves the overload at compile time. The linker does not need to decide which `add` we intended: the two calls already refer to different symbols.

When our application called both functions, its **Procedure Linkage Table (PLT)** contained corresponding entries:

```text
<_Z3addii@plt>
<_Z3adddd@plt>
```

On this Linux build, those named stubs were in the `.plt.sec` section. The PLT is an ELF linking mechanism, not a universal feature of every executable format.

### The return type is not necessarily in the name

We then changed:

```cpp
int add(double a, double b);
```

to:

```cpp
double add(double a, double b);
```

The exported symbol remained:

```text
_Z3adddd
```

For these ordinary non-template functions under this ABI, the return type is not encoded in the mangled name. C++ also does not allow overloading solely by return type.

That does **not** mean the change is binary-compatible. On our Linux x86-64 target, an `int` result is returned in `eax`, while a `double` result is returned in `xmm0`. Preserving the symbol name does not preserve the return-value convention.

For this particular check, we rebuilt both sides and inspected the symbol; we did not run an old caller against the changed return type.

## Putting the function inside a class

Next, we moved the operations into a class:

```cpp
class Calculator {
public:
    int add(int a, int b);
    double add(double a, double b);
};
```

The exported names became:

```text
_ZN10Calculator3addEii  -> Calculator::add(int, int)
_ZN10Calculator3addEdd  -> Calculator::add(double, double)
```

The class name is now part of the encoded identity. Because these are non-static member functions, calls also pass an implicit `this` pointer.

So far, we have only observed one compiler's conventions. Does another compiler make the same choices?

## Three compilers, but not three fixed ABIs

For the cross-compiler experiment, we used **Windows x64** so all three toolchains could target the same operating system and executable format:

- GCC 13.2 through MinGW-w64
- Clang 22.1 through `clang-cl`, targeting the Microsoft environment
- MSVC 19.44

Each toolchain built a DLL containing the same two `Calculator::add` methods. We then attempted to build an application with each compiler against each DLL.

The application called both overloads to check whether the compiled caller and DLL could link and run together.

| Application compiler / Library compiler | GCC / MinGW | Clang / Microsoft target | MSVC |
|---|---|---|---|
| **GCC / MinGW** | Passed | Link failed | Link failed |
| **Clang / Microsoft target** | Link failed | Passed | Passed |
| **MSVC** | Link failed | Passed | Passed |

### Why did some combinations fail?

The exported names explain the first incompatibility:

| Method | GCC / MinGW | MSVC and clang-cl |
|---|---|---|
| `add(int, int)` | `_ZN10Calculator3addEii` | `?add@Calculator@@QEAAHHH@Z` |
| `add(double, double)` | `_ZN10Calculator3addEdd` | `?add@Calculator@@QEAANNN@Z` |

The GCC caller requested GNU-style symbols. The Microsoft-targeted library exported Microsoft-style symbols. The reverse combination had the same disagreement.

These combinations failed during linking, before an application could run.

This is not simply a rule that different compilers cannot interoperate. **Clang and MSVC did interoperate in our test.**

Clang can target different ABI environments. On typical Linux targets, GCC and Clang use the Itanium C++ ABI family. When targeting the Microsoft environment, Clang implements the Microsoft C++ ABI instead. The name “Itanium” does not restrict that ABI to Itanium processors.

The compiler documentation makes these contracts explicit:

- [GCC's ABI policy](https://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html) discusses name mangling, object layout, virtual functions, and ABI-affecting options.
- [Clang's MSVC compatibility documentation](https://clang.llvm.org/docs/MSVCCompatibility.html) describes its Microsoft ABI support and limitations.
- [Microsoft's binary compatibility documentation](https://learn.microsoft.com/en-us/cpp/porting/binary-compat-2015-2017?view=msvc-170) describes compatibility across its toolset versions, with restrictions on linker versions, runtimes, and whole-program optimization inputs.

Our matrix is evidence for this small interface and these configurations, not a guarantee for arbitrary C++ libraries. We did not test exceptions, standard-library objects, or complex inheritance across these DLL boundaries. We also did not run a Linux GCC-versus-Clang matrix.

**The target ABI matters more than the compiler's name alone.**

## Even a private member can break the contract

Suppose we avoid the cross-compiler problem entirely. Both application and library use the same compiler and target.

Can a small internal class change still break compatibility?

Consider this original class:

```cpp
class Calculator {
    int value;

public:
    void set(int x);
    int get() const { return value; }
};
```

The shared library implements:

```cpp
void Calculator::set(int x)
{
    value = x;
}
```

The application creates the object itself:

```cpp
Calculator calculator{};
calculator.set(42);
```

With the original library, our Linux test reported:

```text
sizeof(Calculator) = 4, value = 42
```

Now add a private member before `value`:

```cpp
class Calculator {
    int extra;  // New private member
    int value;

public:
    void set(int x);
    int get() const { return value; }
};
```

No existing public function signature changed. The exported symbol for `set(int)` also remained identical:

```text
_ZN10Calculator3setEi
```

We rebuilt **only the library** and ran the unchanged application. This time:

```text
*** stack smashing detected ***: terminated
```

### The function was found. The object was wrong.

This was not a failure to resolve `set(int)` through dynamic linking. The old application and new library disagreed about memory layout:

| Property | Old application's definition | New library's definition |
|---|---|---|
| Object size | 4 bytes | 8 bytes |
| Offset of `value` | 0 | 4 |

The application reserved storage using the old definition. The library's new `set()` wrote to `value` using the new offset, beyond the original object's storage.

That is undefined behavior. Our build detected stack corruption, but another build could behave differently or silently corrupt memory. An immediate diagnostic is not guaranteed.

Rebuilding the application against the new header restored agreement:

```text
sizeof(Calculator) = 8, value = 42
```

The important observation is:

> **`private` restricts source-level access. It does not hide the class's storage requirements from code that creates its objects.**

This is not unique to C++: changing the layout of a publicly exposed C structure can cause the same category of failure. C++ access control simply makes it tempting to mistake private members for details that cannot affect callers.

## Does PImpl solve this?

The **pointer-to-implementation**, or **PImpl**, pattern addresses exactly this layout exposure.

Instead of placing evolving state directly in the public class, keep it in a separately defined implementation object:

```cpp
class Calculator {
public:
    Calculator();
    ~Calculator();

    Calculator(const Calculator&) = delete;
    Calculator& operator=(const Calculator&) = delete;

    int add(int a, int b);

private:
    class Impl;
    Impl* impl;
};
```

The implementation lives inside the library:

```cpp
class Calculator::Impl {
public:
    int value = 0;
    // More internal state can be added here.
};

Calculator::Calculator() : impl(new Impl) {}
Calculator::~Calculator() { delete impl; }

int Calculator::add(int a, int b)
{
    return a + b; // Real operations can use impl's state.
}
```

This illustrative class disables copying to avoid accidentally sharing and double-deleting the raw pointer. A production implementation would need deliberate ownership and copy/move semantics.

Adding members to `Impl` does not change the public `Calculator` object's layout. Its storage still consists of the implementation pointer. This protects against the particular private-member change we just demonstrated.

But it does not eliminate the rest of the contract:

- Existing public methods still depend on the selected C++ ABI.
- Changing their signatures can still break callers.
- Changing public inheritance or virtual layouts still requires care.
- Exposing types such as `std::string` introduces standard-library ABI dependencies.

PImpl would not make the incompatible symbol names in our GCC/MinGW-versus-MSVC experiment match.

It also has maintenance and runtime costs. A typical heap-backed implementation needs allocation, ownership management, and pointer indirection. Designs that mirror public operations in `Impl` also require forwarding methods and two method surfaces to keep synchronized.

That is extra work, but not necessarily twice the work: the business logic need not be duplicated, and PImpl does not require every public function to have a matching implementation method.

**PImpl protects implementation layout. It does not make every public interface change safe or make incompatible compiler ABIs interchangeable.**

This section describes the pattern; unlike the preceding results, it is not a separate measured experiment.

## To Summarize

We started by looking at function symbols, but ABI compatibility extends well beyond their names.

Our experiments showed that:

- C++ encodes function identity using ABI-specific naming conventions.
- Different compilers can interoperate when they implement compatible binary conventions, as our MSVC and `clang-cl` builds did for this interface.
- The same compiler cannot protect an old application from a changed class layout.
- An unchanged function symbol does not prove that its calling or object-layout contract remains compatible.

PImpl can reduce one important source of coupling, but the public boundary still needs care.

The conclusion is not that C++ cannot maintain ABI compatibility, or that every small change breaks it. It is that **source-level encapsulation and binary compatibility are different properties**.

An old executable does not see our intention to make a harmless private change. It continues using the sizes, offsets, symbols, and calling conventions it was compiled to expect.
