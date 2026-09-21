# ZEP-0021: Projects, Dependencies, and Artifact Targets

**English** | [Traditional Chinese](ZEP-0021-projects-and-artifacts.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0021 |
| **Title** | Projects, Dependencies, and Artifact Targets |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-09-18 |
| **Supersedes** | [ZEP-0017](ZEP-0017-package-manager.md) |
| **Requires** | [ZEP-0020](ZEP-0020-v0.3-core-language.md) |
| **Reference implementation** | `Ryan-2013/zyenlang` tag `v0.3.0` |

## Abstract

ZyenLang 0.3 integrates project creation, dependency locking, target
selection, output placement, and safe cleanup into the flat `zy` command. The
old `zy pkg` family is removed. The first resolver supports local paths and Git
repositories pinned to exact revisions; it deliberately has no registry or
build-script execution.

## Commands

```text
zy new PATH [--lib]
zy init [PATH] [--lib]
zy add ALIAS --path PATH
zy add ALIAS --git URL --rev COMMIT
zy remove ALIAS
zy fetch [--locked]
zy check [TARGET]
zy build [TARGET] [--release] [--out-dir PATH]
zy run [TARGET] -- [ARGS...]
zy test
zy clean [TARGET]
zy metadata
zy emit --file SOURCE --kind c --out-dir PATH [--output-name NAME]
```

`zy check --file SOURCE [--library]` exists for editors and standalone source
inspection. Single-file `zy build source.zy -o output` is removed because an
extension must never select the artifact kind. `zy emit` is the explicit
non-project C-source operation.

`zy run` executes a project binary with the project root as its working
directory, so relative filesystem paths are independent of the shell location
that invoked the command. `zy test` and compiler single-file execution use the
entry source file's directory.

## Manifest and targets

The manifest is `zyproject.toml`:

```toml
[package]
name = "zy-math"
version = "0.3.0"
zyen = ">=0.3.0"

[build]
default-target = "ffi"
target-dir = "target"

[targets.app]
kind = "bin"
entry = "src/main.zy"

[targets.ffi]
kind = "c-source"
entry = "src/lib.zy"
out-dir = "../consumer/generated"
output-name = "zy_math"

[dependencies]
utils = { path = "../utils" }
net = { git = "https://example.com/net.git", rev = "COMMIT_SHA" }
```

Supported kinds are:

- `bin`: a native executable requiring `fn main() i32`;
- `c-source`: C source/header/runtime/native bundle plus metadata;
- `staticlib`: a platform static library and public header;
- `sharedlib`: a platform shared library, public header, and import library
  where the platform requires one.

Default output is `target/debug/<target>/` or
`target/release/<target>/`. Target `out-dir` is resolved from the project root.
CLI `--out-dir` has highest precedence. `output-name` defaults to the target
key. Artifact selection is independent of filename suffixes.

Only `export fn` declarations enter a generated C/C++ header. In 0.3, direct
exports are restricted to non-generic, non-throwing functions using fixed
width numbers, `bool`, `void`, and `ZL_String`. Complex APIs need a supported
facade.

## Dependency graph and imports

A dependency key is its import alias. A path dependency resolves from the
owning manifest. A Git dependency requires an exact revision and records the
resolved commit. A package may import only its direct dependencies:

```zy
import utils::math as math
let answer = math::add(20, 22)
```

Imports stay below the dependency's `src/` root. The resolver rejects path
escape, symlinks, dependency cycles, alias collisions, unsafe names, mutable
Git revision syntax, unsupported URL forms, and configured graph/file/byte
limits.

`zy.lock` records the root manifest digest and, for each package, its alias,
package identity, source, exact commit where applicable, content digest, and
transitive edges. Resolution order is deterministic. `zy fetch` updates the
lock; `zy fetch --locked`, check, build, and run reject a missing, stale, or
content-mismatched lock instead of changing it silently.

Package snapshots are immutable and content addressed under
`~/.zyen/packages/0.3/<sha256>/`. Git source checkouts are stored below
`~/.zyen/git/0.3/` after their repository metadata is removed.

## Tests and metadata

`zy test` runs manifest targets named `test` or beginning with `test-`. If none
exist, it runs each `tests/**/*.zy` executable and stops on a nonzero result.

`zy metadata` emits machine-readable package, target, build, and locked
dependency data without compiling the program.

## Safe cleanup

Every successful build records the exact absolute files it created in an
artifact state file. `zy clean [target]` removes only those files. It rejects
relative paths, malformed records, symlinks, and directories. It never
recursively deletes `out-dir`, including an external directory containing
unrelated consumer files.

## Security and scope

Dependency fetch never runs source or an install hook. Exact revisions and
SHA-256 digests make inputs reproducible and detect drift; they do not make
native dependency code safe. Building or running a package trusts its native
C inputs with the current process privileges.

0.3 has no registry, semver range solver, publishing, build scripts,
dependency hooks, prebuilt native package format, or re-export.
