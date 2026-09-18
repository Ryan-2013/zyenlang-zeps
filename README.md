# ZyenLang Enhancement Proposals (ZEPs)

**English** | [繁體中文](README.zh-TW.md)

> **Related repositories**
> - **Compiler, runtime, stdlib, and VS Code support**: [zyenlang](https://github.com/Ryan-2013/zyenlang)

ZyenLang 0.3 is a hard language break. ZEP-0020 through ZEP-0022 are the
canonical specifications for 0.3. Superseded proposals remain available as
design history and must not be treated as current syntax.

A small, opinionated specification system for ZyenLang, modelled after
Python's PEP system but stripped to what a single-author + AI-assisted
project actually needs.

Each ZEP is one self-contained markdown file. There are three types:

| Type | Purpose | Example |
|---|---|---|
| **Process** | How the project is run | ZEP-0001 (Purpose and Process) |
| **Informational** | A convention, best practice, or style guide | ZEP-0003 (Style Guide) |
| **Standards** | A rule that compiler or stdlib must enforce | ZEP-0004 (Global State) |

ZEPs that codify already-shipped behaviour are marked `Active` (process /
informational) or `Final` (standards). ZEPs that propose changes are
`Draft` until accepted.

## Index

| ZEP | Title | Status | Type |
|---|---|---|---|
| [0001](ZEP-0001-purpose-and-process.md) | Purpose and Process | Active | Process |
| [0002](ZEP-0002-zep-template.md) | ZEP Template | Active | Process |
| [0003](ZEP-0003-style-guide.md) | ZyenLang Style Guide | Active | Informational |
| [0004](ZEP-0004-global-state-and-constants.md) | Global State and Module-Level Constants | Final | Standards |
| [0005](ZEP-0005-error-handling.md) | Error Handling Conventions | Active | Informational |
| [0006](ZEP-0006-struct-field-defaults.md) | Struct Field Defaults | Superseded | Standards |
| [0007](ZEP-0007-zy-doctor.md) | `zy doctor`: System Health Check | Active | Standards |
| [0008](ZEP-0008-zyenv.md) | `zyenv`: ZyenLang Version Manager | Active | Standards |
| [0010](ZEP-0010-first-class-functions.md) | First-Class Function Values | Superseded | Standards |
| [0011](ZEP-0011-struct-body-methods.md) | Struct-Body Method Definitions | Superseded | Standards |
| [0013](ZEP-0013-closures.md) | Closures and Lambda Lifting | Superseded | Standards |
| [0014](ZEP-0014-managed-pointers-and-arc.md) | Managed Pointers and Automatic Reference Counting | Superseded | Standards |
| [0015](ZEP-0015-function-cell-pointers.md) | Managed Function Cell Pointers | Superseded | Standards |
| [0016](ZEP-0016-structural-list-dispatch.md) | Structural List Method Dispatch | Superseded | Standards |
| [0017](ZEP-0017-package-manager.md) | `zy pkg`: Package Manager | Superseded | Standards |
| [0018](ZEP-0018-pointer-postfix-and-owned-fields.md) | Pointer Postfix Precedence and Owned Struct Fields | Superseded | Standards |
| [0019](ZEP-0019-native-modules-portable-toolchain.md) | Native Modules and the Portable C Toolchain | Superseded | Standards |
| [0020](ZEP-0020-v0.3-core-language.md) | ZyenLang 0.3 Core Language Model | Final | Standards |
| [0021](ZEP-0021-projects-and-artifacts.md) | Projects, Dependencies, and Artifact Targets | Final | Standards |
| [0022](ZEP-0022-native-compatibility-abi.md) | Native Compatibility and C ABI v2 | Final | Standards |

## How to read these

For the current language, read ZEP-0020, ZEP-0021, and ZEP-0022 first. Earlier
documents explain the design history; an older rule marked `Superseded` is not
part of ZyenLang 0.3.

If you're writing a new ZEP, copy `ZEP-0002-zep-template.md`, increment
the number, and submit.

## What is *not* a ZEP

- Bug reports — those live in the compiler repo's `docs/` folder.
- Per-module API documentation — those live in `docs/std_<module>.md`.
- Changelogs and version notes belong in the compiler repository.

ZEPs document the *design principles* and *long-lived conventions*.
If a rule fits on one line in the README, it doesn't need a ZEP.
