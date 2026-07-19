# ZEP-0017 - `zy pkg`: Package Manager

**English** | [Traditional Chinese](ZEP-0017-package-manager.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0017 |
| **Title** | `zy pkg`: Package Manager |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-07-19 |
| **Requires** | [ZEP-0008](ZEP-0008-zyenv.md) |
| **See Also** | [ZEP-0019](ZEP-0019-native-modules-portable-toolchain.md) |

## Abstract

This ZEP defines a reproducible package workflow with the convenience of
`pip install`. The package manager is the `zy pkg` command family. It uses a
project manifest, an exact lockfile, and an immutable content-addressed cache.

`zyenv` continues to select compiler versions. It does not install project
libraries.

## Commands

```text
zy pkg init
zy pkg add httpx
zy pkg add github:Ryan-2013/zy-raylib
zy pkg add ../local-library
zy pkg remove httpx
zy pkg install
zy pkg update [package]
zy pkg list
zy pkg publish
```

`zy pkg add` changes both the manifest and lockfile. `zy pkg install` installs
the exact locked graph. `--locked` rejects manifest drift, and `--offline`
rejects any operation that would require network access.

## Manifest

The project manifest is `zyproject.toml`:

```toml
[package]
name = "my-game"
version = "0.1.0"
entry = "src/main.zy"
zyen = ">=0.1.84,<0.2"

[dependencies]
httpx = "^1.2.0"
raylib = { git = "https://github.com/Ryan-2013/zy-raylib", rev = "..." }
local-ui = { path = "../local-ui" }
```

Registry versions follow semantic versioning. Git dependencies are pinned by
commit. Path dependencies are development-only and cannot be published
without replacement.

## Lockfile and resolution

`zy.lock` records exact versions or commits, source URLs, dependency edges,
and SHA-256 archive digests. Resolution selects one version per package name
for the complete project. The first resolver uses deterministic backtracking,
preferring the highest compatible version. A conflict reports the shortest
dependency chains that introduced incompatible constraints.

Only `add`, `remove`, and `update` intentionally change the lockfile. Check,
run, and build consume it without silently selecting newer versions.

## Storage and imports

Archives are immutable and content-addressed:

```text
~/.zyen/packages/v1/<sha256>/
```

The compiler reads `zy.lock` and resolves package imports directly from that
cache:

```zy
import <httpx/client> as http;
```

Relative imports may not escape a package root. A project does not copy
dependency source into its repository.

## Native packages and security

Native packages use the same public `.zlcm.h`, `.h`, and `.c` mechanism as
`std/c_module`. Installation does not execute package code, compiler plugins,
or build scripts. Platform metadata uses the existing `ZLC_SOURCE_*`,
`ZLC_HEADER_*`, `ZLC_LIB_*`, and `ZLC_CFLAG_*` declarations.

Installation safety is not a native-code sandbox. Building a native package
passes its declared sources and permitted flags to the C toolchain, and the
resulting executable has the user's normal process permissions. Package-root
containment, flag policy, symbol validation, compiler invocation, and explicit
native-code acknowledgement follow [ZEP-0019](ZEP-0019-native-modules-portable-toolchain.md).

Registry downloads use HTTPS and are verified against the locked SHA-256
digest. A name/version pair is immutable after publication. Precompiled native
binaries and install-time scripts are outside the first version.

## Delivery

1. Local manifest, path dependencies, lockfile, cache, and compiler resolver.
2. Git dependencies pinned by commit plus offline and locked modes.
3. Read-only registry with add/install/update and conflict diagnostics.
4. Authentication, ownership, publish, yank, and search.

Every phase is tested on Windows, Linux, and macOS, and the portable compiler
must not require a system Python installation or system C compiler.
