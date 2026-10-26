---
title: "Evolving Libraries Without Breaking Old Binaries"
date: 2026-10-26 00:00:00 +0900
categories: [Software Engineering]
tags: [c-language, abi, backward-compatibility, shared-libraries, glibc, windows, sqlite]
description: "How glibc, Windows COM, and SQLite preserve old contracts while adding new features through symbol versioning, immutable interfaces, and opaque handles."
---

Adding a feature is usually straightforward when we control every caller. We change a function, update its users, and rebuild everything.

A public library does not have that luxury.

Some applications were compiled years ago. Their source may be unavailable, their developers may have moved on, and their users still expect them to work after an update.

In the [previous post]({% post_url 2026-10-21-understanding-abi-and-backward-compatibility %}), we explored three contracts that existing applications depend on:

1. **Exported function signatures**
2. **Public data layouts**
3. **API behavior**

We also saw that successful symbol resolution does not prove compatibility. An executable can find the right function name and still pass the wrong arguments or access memory using an outdated structure layout.

So how do mature software projects evolve without breaking those contracts?

Three real-world examples offer complementary answers:

- **glibc** preserves historical entry points through **symbol versioning**.
- **Windows COM** introduces **new interfaces with distinct identities**.
- **SQLite** combines **opaque handles** and **new function names**.

The common principle is simple:

> **Preserve existing contracts while introducing new ones.**

## glibc and the optimization that broke working applications

The glibc `memcpy()` incident demonstrates a difficult distinction: **an application can be incorrect according to the specification and still be important to keep running**.

### The application bug was already there

`memcpy()` copies bytes between **non-overlapping** memory ranges. If the ranges overlap, the application must use **`memmove()`** instead.

Consider:

```c
char buffer[] = "ABCDE";

memcpy(buffer, buffer + 1, 4); /* Undefined behavior: overlap */
```

The programmer probably intended to shift the characters left. That operation is valid with:

```c
memmove(buffer, buffer + 1, 4);
```

But an incorrect `memcpy()` call might appear to work if the implementation happens to copy in a favorable order.

### What changed in glibc 2.13?

A performance optimization changed copying order in some implementations, including on x86-64. This exposed overlapping-buffer bugs that previous implementations had accidentally hidden.

The original bug report describes failures in existing applications, including Flash, and notes that behavior could vary with the CPU-specific implementation selected. [Original glibc bug report](https://sourceware.org/pipermail/glibc-bugs/2011-February/012318.html)

The signature had not changed:

```c
void *memcpy(void *dest, const void *src, size_t n);
```

Arguments still arrived correctly. The exported symbol still existed. Nevertheless, previously working binaries failed.

**This was not a calling-convention ABI break. It was a practical compatibility regression caused by reliance on undefined behavior.**

### The solution in glibc 2.14

The compatibility solution used **symbol versioning**. On x86-64, the old `memcpy` symbol version was associated with an overlap-safe implementation based on `memmove`, while a newer symbol version became the default for newly linked code. [Historical glibc compatibility patch](https://sourceware.org/pipermail/libc-alpha/2011-March/025155.html)

Conceptually:

```text
Old binary -> memcpy@GLIBC_2.2.5 -> compatibility implementation
New binary -> memcpy@GLIBC_2.14  -> current interface
```

Both contracts could coexist inside one shared library.

### What symbol versioning adds

Without symbol versioning, a caller requests a function name:

```text
memcpy
```

With versioning, its dynamic-linking metadata can request a **name and version**:

```text
memcpy + GLIBC_2.2.5
```

At build time, the linker records the selected requirement. At runtime, the dynamic linker resolves that requirement against the installed library.

Tools may display definitions like:

```text
memcpy@GLIBC_2.2.5
memcpy@@GLIBC_2.14
```

The **`@@`** marks the default definition selected for ordinary references when linking new applications. An old executable's recorded requirement does not change merely because the library's default changes.

These version labels are identifiers—not instructions to load an entire old library, and not numbers the loader compares to select the highest version. [GNU linker: Symbol Versioning](https://sourceware.org/binutils/docs/ld/VERSION.html)

The lesson is not that overlapping `memcpy()` became valid. It remains undefined behavior, and compiler optimizations can bypass the library call altogether.

The lesson is that **library maintainers sometimes preserve accidental behavior to protect deployed binaries**.

## Windows COM and immutable interfaces

Windows uses multiple compatibility mechanisms. **COM**, the Component Object Model, provides a particularly clear example of explicit binary contracts.

A COM interface has a unique **Interface Identifier**, or **IID**. Microsoft's rule is that a published interface is **immutable**: its definition must not change. [Microsoft: Interface Design Rules](https://learn.microsoft.com/en-us/windows/win32/com/interface-design-rules)

### Why an interface cannot simply grow arbitrarily

A COM interface pointer provides access to a defined **vtable** containing function pointers.

An illustrative calculator interface might have:

```text
ICalculator vtable
  [0] QueryInterface
  [1] AddRef
  [2] Release
  [3] Add
```

Compiled callers depend on the method slots and their signatures.

Changing `Add` from two arguments to three would not update existing machine code. The caller would continue making the old call—exactly the problem from our shared-library experiment.

### Give the new contract a new identity

Instead, the component can preserve **`ICalculator`** and introduce **`ICalculator2`**, with a different IID:

```text
Old application -> ICalculator  --+
                                 +-> Updated component
New application -> ICalculator2 --+
```

The application can use **`QueryInterface()`** to ask whether the component supports the extended interface.

If supported, it receives an interface pointer. If not, the result is **`E_NOINTERFACE`**, allowing a suitably designed application to fall back. [Microsoft: QueryInterface](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nf-unknwn-iunknown-queryinterface%28refiid_void%29)

**`QueryInterface()` discovers a contract; it does not translate method calls.** Any adaptation happens inside the component.

### Does keeping old interfaces require keeping old code?

Not necessarily.

An old entry point can be a compatibility wrapper around shared internals:

```c
/* Illustrative implementation, not Windows source code. */
static int add_impl(int a, int b, int c)
{
    return a + b + c;
}

int add_old(int a, int b)
{
    return add_impl(a, b, 0);
}

int add_new(int a, int b, int c)
{
    return add_impl(a, b, c);
}
```

The old interface might delegate to the new operation, or both might call a common implementation.

When the contracts differ substantially, separate compatibility behavior may be necessary. Sharing code is safe only when the old results, errors, side effects, and ownership rules are preserved.

> **The obligation is to preserve old contracts—not necessarily old code.**

### Compatibility also needs sustained testing

Interface rules prevent some mistakes, but cannot establish that every application still behaves correctly.

Microsoft has described automated integration gates, broader application tests, manual verification, and telemetry-based regression detection. Its 2019 account reported over **280,000 application-compatibility tests per day**; that is a historical figure, not a statement about today's infrastructure. [Microsoft: Application Compatibility Testing](https://blogs.windows.com/windowsexperience/2019/01/15/application-compatibility-in-the-windows-ecosystem/)

For our own library, the essential test is:

```text
Old executable + new library
```

Recompiling the application tests a different question.

For a COM-style component, useful regression cases include checking that the original IID is still supported, exercising the original methods and error paths, and verifying reference-counted lifetime. New clients should also test their fallback behavior against components without the new interface. These are suggested test cases, not a description of a specific Microsoft internal suite.

Windows can also apply targeted **compatibility shims** to particular applications, adjusting calls without imposing that behavior on every process. [Microsoft: Application Compatibility Database](https://learn.microsoft.com/en-us/windows/win32/devnotes/application-compatibility-database)

## SQLite and interfaces designed to hide change

SQLite illustrates how much compatibility can come from **avoiding unnecessary exposure in the first place**.

### Opaque handles protect internal layouts

In our earlier experiment, the application allocated a public structure. When the replacement library expected a larger object, it read beyond the allocation.

SQLite's database connection instead has this public declaration:

```c
typedef struct sqlite3 sqlite3;
```

The application can hold a **`sqlite3 *`**, but cannot inspect the object's members or compute its size from that declaration.

It uses functions such as:

```c
sqlite3 *db = NULL;

int rc = sqlite3_open("example.db", &db);
if (rc == SQLITE_OK) {
    /* Use SQLite APIs to operate on db. */
}

if (db != NULL)
    sqlite3_close(db);
```

SQLite manages the object behind the opaque handle. [SQLite: Database Connection Handle](https://sqlite.org/c3ref/sqlite3.html)

```text
Application knows: pointer type and API contract
Library knows:     allocation, layout, and internal state
```

**An opaque handle is not necessarily a function-pointer table.** The important property is that callers do not depend on its private layout.

That gives the implementation freedom to add internal fields without changing member offsets embedded in application binaries. The handle still belongs to the library instance that created it; opacity does not make objects interchangeable between separately loaded library instances.

### New entry points preserve old behavior

SQLite's statement-preparation family shows another technique:

```text
sqlite3_prepare()     Legacy interface
sqlite3_prepare_v2()  Newer behavioral contract
sqlite3_prepare_v3()  Additional preparation flags
```

The `_v2` interface retains the original parameter shape but changes aspects of subsequent statement behavior. The `_v3` interface adds a flags argument under a new name.

Older interfaces remain available for backward compatibility. SQLite recommends **`sqlite3_prepare_v2()`** for ordinary use, with `_v3()` available when its flags are needed. [SQLite: Statement Preparation APIs](https://sqlite.org/c3ref/prepare.html)

The key distinction is:

> **A new name can version semantics, not just a function signature.**

### But will the function list keep growing?

Yes. Retaining old functions while adding new contracts expands the public interface.

Not every feature needs a new function: internal improvements, new SQL capabilities, or existing flags may provide extension points. But incompatible interface changes often require new entry points.

New developers can still choose legacy APIs. Documentation, reviews, and project-specific lint rules can discourage that choice; the linker does not know whether a caller is newly written.

**Deprecation and removal are different decisions.**

- **Deprecation** discourages future use.
- **Removal** withdraws an entry point that existing binaries may still require.

An API can remain deprecated for a long time. SQLite specifically describes the original preparation API as legacy and discourages its use. Even if no new application adopts an old API, old applications can keep the maintenance obligation alive.

## Three approaches with one shared principle

| Project | Technique | What remains stable |
|---|---|---|
| **glibc** | Versioned symbols | A historical function contract |
| **Windows COM** | Immutable interfaces with distinct IIDs | Method layout and interface meaning |
| **SQLite** | Opaque handles and new entry points | Public function contracts and opaque object boundaries |

None of these techniques automatically proves compatibility.

A version label cannot verify behavior. An IID cannot prevent a faulty implementation. An opaque pointer does not excuse changing ownership rules.

Design must be supported by **regression tests, old-binary testing, and clear compatibility policies**. SQLite, for example, documents extensive regression, fault-injection, fuzzing, and dynamic-analysis testing. Those practices validate behavior; they do not replace ABI discipline. [SQLite: How SQLite Is Tested](https://sqlite.org/testing.html)

These promises also have boundaries: a compatible platform ABI and build configuration are still required. Database-file compatibility, source compatibility, and binary compatibility are separate questions.

## Compatibility is a maintained boundary

Modern libraries do not preserve compatibility by refusing to change. They **control where change becomes visible**.

Internals can evolve aggressively when applications do not depend on them. New contracts can introduce new capabilities while old contracts remain available.

Sometimes an adapter is enough. Sometimes compatibility requires separate behavior. Eventually, a project may choose an explicitly incompatible release—but that is a support decision, not something symbol resolution can make harmless.

The practical rules are:

1. **Expose only what callers genuinely need.**
2. **Give incompatible contracts distinct identities.**
3. **Reuse implementation without changing old semantics.**
4. **Treat retained interfaces as ongoing obligations.**
5. **Test old binaries against new implementations.**

**Backward compatibility is not a feature completed once. It is a boundary that must be maintained every time the software changes.**
