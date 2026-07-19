# ZEP-0019 - Native Modules and the Portable C Toolchain

**English** | [Traditional Chinese](ZEP-0019-native-modules-portable-toolchain.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0019 |
| **Title** | Native Modules and the Portable C Toolchain |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-07-19 |
| **See Also** | [ZEP-0007](ZEP-0007-zy-doctor.md), [ZEP-0017](ZEP-0017-package-manager.md) |

## Abstract

ZyenLang transpiles to C, so native interoperability is part of the normal
build pipeline rather than a separate foreign runtime. This ZEP specifies C
compiler selection, the contents of a portable release, the compile-time
behaviour of `std/c_module`, the bundled `std/tk` GUI backend, and the trust
boundary of native packages.

A portable release includes its own Zig C compiler and platform GUI runtime.
It must work after extraction and adding its command directory to `PATH`,
without a separately installed Python, GCC, or Raylib development package.

## C compiler selection

Commands that produce or run a native executable select a compiler in this
order:

1. The command named by `ZY_CC`, when set.
2. The Zig executable bundled under the ZyenLang `toolchain/` directory,
   invoked as `zig cc`.
3. A system `gcc`, `clang`, or `cc`, in that order.

The first usable candidate is selected. `zy doctor` must use this same
resolver and report both the compiler identity and whether it is bundled or
system-provided. It must not report a missing compiler merely because `gcc`
is absent when bundled Zig cc is usable.

Zig cc is only the C backend. ZyenLang source is not translated to Zig and
does not depend on the Zig language or package manager.

## Native build pipeline

`c_module.load()` is a compile-time declaration. It does not generate a
runtime loader call and does not invoke a Python wrapper generator.

For `zy run` and executable builds, the compiler performs these stages:

1. Resolve `.zy` imports and each literal `.zlcm.h` or `.zlcm.json` template.
2. Generate an internal ZyenLang wrapper type for every distinct resolved
   template.
3. Transpile the expanded ZyenLang program to C.
4. Invoke the selected C compiler with the generated C file and every
   `ZLC_SOURCE` source in the same compile/link command.
5. Force-include declared headers, add include and library directories, then
   apply the selected platform libraries and flags.
6. Link one native executable.

Headers are included, not compiled as independent translation units.
Prebuilt libraries declared by `ZLC_LIB` are linked at the final stage.

`zy check` parses templates, expands wrappers, and type-checks calls, but does
not invoke the C compiler. A C-only build emits the generated `.c` file but
does not compile native sources. Native compiler diagnostics therefore occur
only when producing an executable.

## Portable release contents

Every supported portable archive is platform-specific and contains:

- the `zy` command, compiler runtime, and standard library;
- a bundled Zig toolchain usable as Zig cc;
- the public `zyenlang_c_abi.h` header;
- the `std/tk` ZyenLang facade and C compatibility source;
- the platform Raylib shared library used by `std/tk`.

The GUI shared library is `raylib.dll` on Windows, `libraylib.so` on Linux,
and `libraylib.dylib` on macOS. `std/tk` may locate it through
`ZYENLANG_RAYLIB`, the portable archive, the executable directory, or the
platform library search path.

Portable means that no separate compiler, Python installation, or Raylib
development package is required. It does not replace operating-system ABI,
window-system, graphics-driver, or C-library facilities. A release for one OS
or CPU architecture is not a release for another.

`std/tk` uses the public `std/c_module` mechanism. Standard-library native
modules receive no private import syntax or compiler-only loading privilege.
They are trusted because they are shipped and tested as part of the signed or
checksummed ZyenLang release.

## Type names and collision resistance

The public type remains `c_module.Module` at the source declaration. The
compiler derives the concrete hidden wrapper type from the canonical template
path and a stable SHA-256 path digest. Consequently:

- two loads of the same resolved template reuse one type and one metadata set;
- two different paths with the same `ZLC_MODULE` name remain distinct;
- hidden names never appear in normal user diagnostics.

All module, function, struct, field, and parameter names read from a template
must be validated before generated source is constructed. Names must be plain
C/ZyenLang identifiers and must not contain whitespace, control characters,
newlines, comments, punctuation, or source fragments. The compiler maintains
a reserved set for entry-point, ABI-runtime, and generated internal symbols.

Path hashing prevents accidental wrapper collisions. It is not a security
sandbox and cannot make arbitrary native C safe.

## Native-code trust boundary

A `.zlcm.h` template and every `.h`, `.c`, library, compiler flag, and linker
flag it selects are native code inputs. A native module can use OS APIs, read
or modify files available to the process, access the network, start threads,
or terminate the program. C constructors may run when the resulting
executable starts, before ZyenLang `main`.

Therefore:

- local native modules are trusted at the same level as C source added to the
  application directly;
- downloading a package must not execute it, but building or running a native
  package crosses the native trust boundary;
- registry packages are verified against the exact SHA-256 digest in
  `zy.lock`;
- registry package paths may not escape their immutable package root through
  absolute paths, `..`, symlinks, source paths, include paths, or library
  paths;
- registry compiler and linker flags use a conservative allowlist;
- flags or external paths outside that policy require an explicit local
  unsafe-native opt-in and may not be published as portable packages;
- compiler commands are passed as argument arrays and never concatenated into
  a shell command;
- third-party native packages require an explicit native-code acknowledgement
  on first addition, unless already trusted by policy or lockfile state.

Locking and hashing prove which bytes were selected; they do not prove those
bytes are harmless. Native packages must never be described as sandboxed.

## Current implementation status

| Requirement | Status |
|---|---|
| Bundled Zig cc preferred over system compilers | Implemented |
| Generated C and `ZLC_SOURCE` files compiled and linked together | Implemented |
| `c_module.load()` erased at compile time | Implemented |
| Path-hashed hidden wrapper types and metadata deduplication | Implemented |
| Cross-platform `std/tk` compatibility source | Implemented |
| Raylib runtime included in each portable release | Implemented |
| Strict validation of every template-provided symbol | Partial |
| Package-root containment, flag policy, and native acknowledgement | Pending with ZEP-0017 |

This ZEP remains `Draft` until the security rows are implemented and covered
by adversarial tests.

## Required tests

- Build and run a basic program from a portable archive with no system C
  compiler on `PATH`.
- Build and open an `std/tk` window using only the bundled GUI runtime.
- Verify compiler selection for `ZY_CC`, bundled Zig cc, GCC, Clang, and no
  usable compiler.
- Verify that same-name modules in different paths do not collide and that a
  repeated template is emitted once.
- Reject control characters, source fragments, reserved symbols, path escape,
  unsafe flags, and symlink escape before invoking the C compiler.
- Verify that `zy check` never compiles or executes native source.
- Test portable archives independently on Windows, Linux, and macOS.

## Copyright

This document is placed in the public domain.
