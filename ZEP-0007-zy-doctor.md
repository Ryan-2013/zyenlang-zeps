# ZEP-0007 — `zy doctor`: System Health Check

**English** | [繁體中文](ZEP-0007-zy-doctor.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0007 |
| **Title** | `zy doctor`: System Health Check |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Draft |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

A read-only health-check command that diagnoses common installation,
environment, and configuration problems for a ZyenLang user. Modelled
after `flutter doctor` and `brew doctor`. Implemented in Python and
shipped as a subcommand of the `zy` CLI. **Always safe to run** — it
never modifies user state.

## Motivation

When `zy build` or `zy run` fails, the cause is almost always one of:

- `gcc` missing or not on `PATH`
- Python version below the minimum
- `tkinter` missing (only matters for the GUI IDE)
- The two `std/` mirrors out of sync
- An installed `zyenlang` package that is stale relative to the source tree
- A path / permissions issue in the working directory

Each of these has an obvious diagnosis, but the user has to *know* to
check for it. `zy doctor` collapses the diagnosis into a single command
that any user can run as their first step when something is wrong.

It also serves as the **bootstrap layer** for the future tools planned
in the ZyenLang ecosystem (`zyenv` version management, `zypm` package
manager): both will assume `zy doctor` passes before attempting
network operations.

## Specification

### Command

```
zy doctor [-v | --verbose] [--json] [--no-runtime]
```

### Categories and checks

Checks are grouped into four categories, run in priority order
(environment first because nothing works without it; runtime last
because it is the most expensive).

#### Environment

| ID | What it checks | Severity if failed |
|---|---|---|
| `python_version` | Python interpreter ≥ 3.8 | **error** |
| `tkinter` | `import tkinter` succeeds | **warning** (CLI works without it) |
| `cc` | A C compiler is on `PATH` (gcc preferred, clang accepted) | **error** |
| `cc_compiles` | The C compiler produces a working `a.out` from `int main(){return 0;}` | **error** |
| `zy_on_path` | The `zy` command is on `PATH` | **warning** (user may invoke via `python -m zyenlang`) |

#### Installation

| ID | What it checks | Severity if failed |
|---|---|---|
| `package_installed` | `import zyenlang` succeeds | **error** |
| `package_version` | Reports the installed version | informational |
| `transpiler_importable` | `from zyenlang import transpiler` succeeds | **error** |
| `std_modules_count` | `zyenlang/std/` contains ≥ 40 `.zy` modules | **error** |
| `std_mirror_sync` | Outer `std/` and inner `zyenlang/std/` are byte-identical | **warning** |

#### Configuration

| ID | What it checks | Severity if failed |
|---|---|---|
| `vscode_ext` | VS Code extension installed | informational |
| `zed_support` | Zed editor support installed | informational |
| `tmpdir_writable` | The session tempdir (e.g. `ide_session/`) is writable | **warning** |

#### Runtime (skipped by `--no-runtime`)

| ID | What it checks | Severity if failed |
|---|---|---|
| `smoke_transpile` | A minimal `.zy` program transpiles to `.c` | **error** |
| `smoke_compile` | The emitted `.c` compiles to an executable | **error** |
| `smoke_run` | The executable runs and prints the expected output | **error** |

### Severity levels

- **informational** — never fails the run, used for version reporting.
- **warning** — does not block use of the toolchain but should be fixed.
- **error** — at least one core feature is broken.

### Human output format

```
ZyenLang doctor — v0.1.49

Environment
  ✓  python 3.11.5
  ✓  tkinter
  ✓  gcc 13.2.0
  ✓  zy on PATH

Installation
  ✓  zyenlang 0.1.49 installed
  ✓  transpiler importable
  ✓  43 stdlib modules
  !  std/ and zyenlang/std/ differ in 1 file:
       std/mem.zy missing alloc_any()
     Fix: cp zyenlang/std/mem.zy std/mem.zy

Configuration
  ✓  VS Code extension installed
  -  Zed support not installed (optional)
     Hint: python tools/install_zed_support.py

Runtime
  ✓  smoke transpile
  ✓  smoke compile
  ✓  smoke run

Summary: 11 passed, 1 warning, 0 errors
```

Symbols: `✓` pass, `!` warning, `✗` error, `-` informational only.

### JSON output format

`--json` emits one machine-readable object per check plus a final
summary, suitable for CI gating and IDE status panels:

```json
{
  "doctor_version": 1,
  "zyenlang_version": "0.1.49",
  "checks": [
    {
      "id": "python_version",
      "category": "environment",
      "status": "pass",
      "detail": "3.11.5"
    },
    {
      "id": "std_mirror_sync",
      "category": "installation",
      "status": "warning",
      "detail": "std/mem.zy missing alloc_any()",
      "fix": "cp zyenlang/std/mem.zy std/mem.zy"
    }
  ],
  "summary": { "pass": 11, "warning": 1, "error": 0 }
}
```

### Exit codes

| Code | Meaning |
|---|---|
| **0** | All checks pass (informational-only checks do not affect exit code). |
| **1** | At least one warning, no errors. |
| **2** | At least one error. |

This matches ZEP-0005 (errors are values): a tool calling `zy doctor`
in CI can branch on the exit code without parsing output.

### Flags

| Flag | Effect |
|---|---|
| `-v`, `--verbose` | Show paths, versions, and extra context for every check. |
| `--json` | Emit JSON instead of human-readable output. Implies non-zero exit on warning/error as above. |
| `--no-runtime` | Skip the smoke-test trio (saves ~3 s; useful in CI doctor passes). |

### What `zy doctor` does **not** do

- **No `--fix` flag in v1.** Doctor reports; the user runs the fix.
  See *Rationale* below.
- **No network checks.** Reachability of GitHub / future `zypm`
  registry is the job of those tools, not doctor.
- **No project-level checks.** Doctor is about the toolchain, not the
  user's `.zy` source. (`zy check <file>` exists for that.)
- **No plugins / custom checks.** The check set is fixed and
  authoritative; an unknown failure should be reported as a bug
  against this ZEP, not patched in user-side code.

## Rationale

### Why Python and not ZyenLang

`zy doctor` is the first command a user runs when ZyenLang is broken.
If doctor itself were a ZyenLang program, a broken ZyenLang would
mean doctor can't start. Python is the bootstrap layer — same reason
`rustup` was not written in Rust until much later in Rust's lifecycle,
and `pyenv` is not written in Python.

This decision is **not** reversible by a future ZEP unless a path
exists to pre-build doctor as a native binary on every supported
platform.

### Why read-only (no `--fix` in v1)

A `--fix` flag would let doctor modify user state (overwrite files,
install packages). That elevates blast radius from "report" to
"destructive action that needs confirmation." We deliberately keep v1
read-only and document the fix for each warning/error so the user can
copy-paste it. A `--fix` flag may come in a later ZEP after the report
side is battle-tested.

### Why the smoke-run check

Static checks (file exists, import works, version meets minimum) can
miss subtle environment problems: wrong libc version, gcc with broken
codegen, antivirus blocking executable launches on Windows. The
smoke trio (transpile → compile → run) is the only check that
exercises the *full pipeline end-to-end*. It catches what the static
checks cannot.

The runtime check is also the one most likely to be skipped in CI for
performance, hence the `--no-runtime` flag.

### Why categorize checks

A flat list of 14 checks is hard to skim. Grouping them by *what
layer of the stack* they verify lets the user scan in priority order:
environment first, runtime last. The mental model mirrors the
dependency graph — fixing an environment error often clears the
installation errors below it, so it should be reported first.

### Why minimum Python 3.8

The walrus operator (3.8) is already used in the transpiler. Dropping
below 3.8 means rewriting hot paths. Python 3.7 reached upstream EOL
in 2023; 3.8 is the practical floor for new development. Doctor
treats below-3.8 as a hard error so we don't get bug reports from
incompatible environments.

### Why categorize `tkinter` as warning, not error

The CLI editor (`ide.zy`) does not need `tkinter`. The GUI editor
(`ide_gui.zy`) does. We can't know which the user intends to use, so
we warn instead of error. The warning text mentions which subcommand
needs the missing dependency.

## Backwards compatibility

Fully backwards-compatible. `zy doctor` is a new subcommand. No
existing behaviour is changed.

The exit-code policy (0 / 1 / 2) is new but only affects scripts that
explicitly call `zy doctor`. No such script exists today.

## Reference implementation

To be added once ZEP-0007 is implemented:

- `zyenlang/cli/doctor.py` — main entry point, output formatter
- `zyenlang/cli/doctor_checks.py` — one function per check ID, each
  returning a `CheckResult` dataclass (id, category, status, detail,
  fix)
- `zyenlang/__main__.py` — wires `zy doctor` as a subcommand
- `tests/test_doctor_python.py` — Python-level unit tests that
  fabricate broken environments and assert the right checks fail

The fixtures should include at least: missing gcc, mirror drift,
wrong Python version (mocked), and an intentionally broken smoke
program (to verify the smoke check itself fails noisily).

## Open questions

- Whether to add a `zy doctor --fix` flag in a future ZEP. Currently
  out of scope to keep v1 surface small.
- Whether to support a user-supplied JSON config that disables
  specific checks (e.g. `--skip vscode_ext` in CI). Possible v1.1
  addition; not in this ZEP.
- Whether the `zy_on_path` check should also detect mismatched
  versions (user has `zy` from a different install on `PATH`).
- Whether to emit color in human output. Default: yes when stdout is
  a TTY, no when piped or `--json`.

## References

- `flutter doctor` — the closest model in feature surface.
- `brew doctor` — secondary inspiration, particularly for the
  warning-vs-error distinction.
- [ZEP-0005](ZEP-0005-error-handling.md) — the int-code convention
  doctor's exit codes follow.
- [ZEP-0001](ZEP-0001-purpose-and-process.md) — status definitions.

## Copyright

This document is placed in the public domain.
