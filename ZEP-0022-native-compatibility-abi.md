# ZEP-0022: Native Compatibility and C ABI v2

**English** | [Traditional Chinese](ZEP-0022-native-compatibility-abi.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0022 |
| **Title** | Native Compatibility and C ABI v2 |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-09-18 |
| **Supersedes** | [ZEP-0019](ZEP-0019-native-modules-portable-toolchain.md) |
| **Requires** | [ZEP-0020](ZEP-0020-v0.3-core-language.md), [ZEP-0021](ZEP-0021-projects-and-artifacts.md) |
| **Reference implementation** | `Ryan-2013/zyenlang` tag `v0.3.0` |

## Abstract

ZyenLang compiles to C11, so C interoperability is a typed build-time
boundary, not an unrelated runtime foreign-function system. This ZEP defines
ordinary native declarations, declarative `.zlcm.h` compatibility templates,
the callback/string ABI, toolchain metadata, and the C/C++ artifacts consumed
by another build system.

Standard-library native modules and third-party packages use the same public
mechanisms. The standard library has no private C loading syntax.

## Direct native declarations

A module may declare package-contained C inputs and typed symbols:

```zy
native source "bridge.c"
native link windows "user32"
private native fn open_native(path: str) i32 = "bridge_open"
```

Native paths are relative to the declaring module and cannot escape its
package root. Source declarations accept C source, not an arbitrary executable
or install script. Native symbols are checked against their ZyenLang
declaration but remain trusted C code at link and runtime.

## Compiler-native compatibility templates

The source interface is:

```zy
import std::c_module as c_module

let math: c_module::Module = c_module::load("native/math.zlcm.h")
let answer: i32 = math.add(20, 22)
```

`c_module::Module` is a dependent compile-time type. The alias used by the type
and initializer must refer to the same `std::c_module` import. A load may
directly initialize a local or a struct/class field default only. The template
path is a string literal resolved relative to the declaring `.zy` file and
must remain inside its package. A bare module type cannot be a parameter or
return because no initializer exists to determine its template.

Loading emits no runtime call and starts no Python wrapper process. The module
graph loader parses the template, creates a path-hashed hidden wrapper, and
adds validated sources, headers, include/library paths, libraries, and flags
to the artifact builder. The same resolved path reuses one hidden type and
metadata record. Different paths remain different types even when they use the
same `ZLC_MODULE` name. Hidden names must not appear in ordinary diagnostics.

The removed `import c_module.load("...") as name` form receives a migration
error showing the import plus direct declaration syntax.

## Template macros

```text
ZLC_MODULE(name)
ZLC_HEADER("relative/header.h")
ZLC_SOURCE("relative/source.c")
ZLC_INCLUDE_DIR("relative/include")
ZLC_LIB_DIR("relative/lib")
ZLC_LIB("library")
ZLC_CFLAG("one-argument")
ZLC_LDFLAG("one-argument")

ZLC_STRUCT(Name, ZLC_FIELD(field, Type), ...)
ZLC_FN(zy_name, c_symbol, ReturnType, ZLC_PARAM(name, Type), ...)
```

Path and build macros may use `_WINDOWS`, `_LINUX`, `_MACOS`, or `_UNIX`.
`_UNIX` applies to Linux, macOS, and other Unix hosts. One CFLAG/LDFLAG macro
contains exactly one argument. Output-changing and response-file flags are
rejected. The toolchain always receives an argument vector, never a
template-constructed shell command.

Template types are fixed-width numbers, `usize`, `f32/f64`, `bool`, `str`,
return-only `void`, `fn(P...) R`, and declared `ZLC_STRUCT` values. `int`,
`float`, and `ZL_String` are compatibility aliases for `i32`, `f64`, and
`str`. `List<T>` and raw pointer values are not stable ABI types in 0.3;
native resources must be hidden behind functions or an opaque handle class.

## C ABI v2

`zyenlang_c_abi.h` defines the public value boundary. `ZL_String` is an
immutable UTF-8 value with retain, release, data, and byte-length operations.
A ZyenLang string returned through an exported function transfers the owned
result required by the generated header contract.

Every first-class function uses:

```c
typedef struct ZL_Function {
    void* call;
    void* env;
    ZL_ArcControl* owner;
    const char* signature;
} ZL_Function;
```

The ABI exposes `zl_fn_retain`, `zl_fn_release`, `zl_fn_assign`,
`zl_fn_clear`, `zl_fn_is_none`, `zl_fn_matches`, and `ZL_FN_CALL_AS`.
Callback parameters are borrowed for the native call. C code that stores one
must retain it with `zl_fn_assign` and clear or replace it with the matching
helper. A returned callback carries an owned reference. Signature validation
uses the exact canonical function type.

Raw C callback APIs, including APIs with no `user_data`, are adapted by the C
compatibility source. The compiler does not synthesize arbitrary library-
specific trampolines.

## Toolchain and artifacts

Native builds select a compiler in this order:

1. `ZY_CC` when explicitly configured;
2. bundled `toolchain/zig[.exe] cc`;
3. system GCC, Clang, or `cc`.

`ZY_CFLAGS` and `ZY_LDFLAGS` add explicit build arguments. `zy check` parses
and validates native metadata but does not invoke a C compiler or execute C.
`zy build` and `zy run` compile native sources with generated C and therefore
cross the trusted-code boundary.

A `c-source` target emits generated source, a C11/C++ header using
`extern "C"`, runtime/ABI headers, declared native sources and local headers,
and metadata describing sources, headers, include/library paths, flags,
libraries, and exports. Static/shared targets compile the same inputs.

## Native GUI ownership

`std::gui::Application` is the public owner of a native window session.
Retained widgets are created through that instance, for example
`app.label(...)`, `app.button(...)`, and `app.column(...)`. Each widget retains
its Application and its `draw()` operation targets only that Application.
Detached module factories such as `gui::label(...)` do not exist.

Raw drawing operations are likewise Application methods. Drawing through a
widget after its Application has closed returns an error instead of using an
implicit process-global target. The current Raylib compatibility layer permits
one open Application per process and rejects a concurrent second open. ARC
cleanup of the final Application reference closes a still-open native session.

The source API is backend-neutral and exposes no Raylib handle. Raylib is the
initial compatibility implementation, not a source-level contract. The next
native backend target is SDL3 for windowing, events, text input/IME, and audio,
with SDL_GPU for rendering. Optional Dear ImGui tooling may live behind this
boundary; application code continues to use the retained Application/widget
API.

## Validation and trust

The compiler rejects unknown/malformed macros, duplicate names, invalid C
identifiers, unsupported or recursive ABI values, conflicting symbol
signatures, missing files, package escape, invalid library names, and unsafe
flags. These checks protect compiler command structure and type agreement.

They do not sandbox C. A native source can access memory, files, the network,
and process APIs with the application's privileges. Dependency review remains
required. Precompiled wrappers built against older by-value ABI layouts must
be rebuilt from source for ABI v2.

The `v0.3.0` delivery is a source tag only. Portable archives, MSI packages,
and GitHub Release assets are a separate verification and release step.
