# ZyenLang Enhancement Proposals (ZEPs)

**English** | [繁體中文](README.zh-TW.md)

> **Related repositories**
> - **Compiler & stdlib**: [zyenlang-v0.1.49](https://github.com/Ryan-2013/zyenlang-v0.1.49)
> - **IDE (dogfood)**: [zyenlang-ide](https://github.com/Ryan-2013/zyenlang-ide)

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
| [0006](ZEP-0006-struct-field-defaults.md) | Struct Field Defaults | Final | Standards |
| [0007](ZEP-0007-zy-doctor.md) | `zy doctor`: System Health Check | Draft | Standards |

## How to read these

If you're new to ZyenLang, read them in order: 0001 explains the system,
0002 shows what a ZEP looks like, 0003 covers everyday style, 0004
explains why there are no top-level variables, and 0005 covers how
errors flow through the stdlib.

If you're writing a new ZEP, copy `ZEP-0002-zep-template.md`, increment
the number, and submit.

## What is *not* a ZEP

- Bug reports — those live in the compiler repo's `docs/` folder.
- Per-module API documentation — those live in `docs/std_<module>.md`.
- Changelogs and version notes — those go in `SPEC_v0_1.md` and release
  notes.

ZEPs document the *design principles* and *long-lived conventions*.
If a rule fits on one line in the README, it doesn't need a ZEP.
