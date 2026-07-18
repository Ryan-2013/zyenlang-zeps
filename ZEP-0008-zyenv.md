# ZEP-0008 — `zyenv`: ZyenLang Version Manager

**English** | [繁體中文](ZEP-0008-zyenv.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0008 |
| **Title** | `zyenv`: ZyenLang Version Manager |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Standards |
| **Created** | 2026-06-21 |
| **Post-History** | 2026-06-21 |

## Abstract

A command-line tool that lets a single user keep multiple ZyenLang
versions installed side by side and switch the active one
per-shell / per-directory / per-machine. Modelled after
`pyenv` / `rbenv` / `nvm`. Implemented in Python and shipped as the
`zyenv` console script of the `zyenlang` package.

## Motivation

Pinning a project to a specific ZyenLang version is a recurring
need:

- Reproducing a bug filed on an older version without losing the
  current installation.
- Comparing language behaviour across versions side by side
  (e.g. `v0.1.43` vs `v0.1.49`).
- A project that builds on stable v0.1.49 while a side experiment
  needs an unreleased dev build.

Before `zyenv`, the only path was to `pip install` one version at a
time and uninstall it before installing the next. The "switch" is
slow, lossy, and a hidden source of "works on my machine".

`zyenv` collapses this into:

```
zyenv install-local D:\path\to\zyenlang-v0.1.43
zyenv install-local D:\path\to\zyenlang-v0.1.49
zyenv use v0.1.49        # global default
cd my_legacy_project
zyenv local v0.1.43      # this dir pins to v0.1.43
```

A shim in `PATH` makes `zy`, `zyen` invocations route to whichever
version `zyenv` resolves for the current shell + cwd.

## Specification

### Layout

```text
$ZYENV_HOME             default: ~/.zyenv
├── versions/
│   ├── v0.1.43/
│   │   └── venv/       full Python venv with zyenlang installed
│   ├── v0.1.49/
│   │   └── venv/
│   └── ...
├── shims/
│   ├── zy.bat          Windows shim (Unix: `zy`, no extension)
│   ├── zyen.bat
│   └── zy_resolver.py  pure-stdlib Python resolver (no zyenlang dep)
└── version             optional single-line: global default version
```

A `.zy-version` file in any project directory (or any ancestor of
the cwd) overrides the global default. `ZY_VERSION` env var
overrides everything for one shell.

### Subcommands

| Command | Behaviour |
|---|---|
| `zyenv init` | Create `~/.zyenv/{versions,shims}`, write the shim, print PATH-edit instructions for the shell. |
| `zyenv list` / `versions` | Print installed versions, marking the active one with `*`. |
| `zyenv current` / `version` | Print active version + the source that selected it (`shell`, `local`, `global`). |
| `zyenv use <version>` | Write the global default to `~/.zyenv/version`. |
| `zyenv local [<version>]` | With arg: write `.zy-version` in cwd. Without arg: print current local pin. |
| `zyenv unset` | Remove `.zy-version` from cwd. |
| `zyenv which` / `path` | Print absolute path of the `zy` binary the shim would exec. |
| `zyenv install-local <path> [--as NAME] [--force]` | Create a fresh venv under `versions/<NAME>/venv` and `pip install -e <path>` into it. `--as` defaults to `vX.Y.Z` read from `pyproject.toml`. |
| `zyenv install-zip <path> [--as NAME] [--force]` | Extract a `.zip` containing a Python package, then install it the same way. |
| `zyenv uninstall <version>` | Remove the version directory; clear the global pointer if it referenced it. |

### Version resolution order

Highest precedence first:

1. `ZY_VERSION` environment variable.
2. Nearest `.zy-version` file walking up from cwd to filesystem root.
3. `$ZYENV_HOME/version` (the global default).

If none resolves, the shim exits with status 1 and a clear message.

### Shim mechanism

The shim is a pure-stdlib Python script
(`~/.zyenv/shims/zy_resolver.py`) that:

1. Reads the resolution order above.
2. Locates `versions/<resolved>/venv/{Scripts,bin}/{zy.exe,zy}`.
3. `subprocess.run`s it with `sys.argv[1:]`, propagates exit code.

On Windows, a tiny `zy.bat` wraps `python "%~dp0\zy_resolver.py" %*`
so the binary name on `PATH` is still `zy`. On Unix, the resolver
itself is the `zy` file with a shebang. Two more files (`zyen.bat` /
`zyen`) duplicate the wrapper for the legacy `zyen` alias.

The shim depends on a system Python being on `PATH` (3.10+). It does
**not** depend on any ZyenLang installation, so a broken installed
version can't take down the shim.

### Installation isolation

Every installed version lives in its own `venv`. There is no shared
site-packages directory across versions. Side effects of installing
one version (e.g. transitive deps, runtime header generation) are
contained.

### Networking

`zyenv install-local` and `install-zip` install with
`pip install --no-build-isolation`. The `venv`'s pre-installed
`setuptools` satisfies the build-system requirement without
contacting pypi.org, so the command works behind SSL-restrictive
networks where pypi cert verification fails. If a target package
declares runtime deps that are not already in the venv, pip will
attempt to fetch them; on restricted networks the user must
pre-stage those via `pip install --target` or accept the install
failure.

## Rationale

### Why per-version venvs (not site-packages directly)

Three reasons:

1. **Cleanup is rm -rf** of one directory. No "pip leftovers".
2. **No site-packages crosstalk.** Two versions can declare
   conflicting C runtime modules without colliding.
3. **The shim has only one job:** point at a fixed binary path. No
   PYTHONPATH gymnastics, no module-resolution surprises.

### Why a Python shim, not a native one

Native shims (a C exe or batch script) need to re-implement
filesystem walk + UTF-8 handling per OS. A Python script in
`stdlib`-only mode is 80 lines and works identically on Windows,
macOS, Linux. The ~50 ms Python startup is unnoticeable for a CLI
tool that itself runs `gcc`.

### Why `--no-build-isolation` by default

PEP 517 build isolation creates a fresh ephemeral venv for the
build, then installs `[build-system].requires` from pypi. On a
restricted network that step fails before any of the user's actual
code is touched. ZyenLang's only build dep is `setuptools >= 68`,
which is exactly what `venv.create` ships. Skipping isolation gives
us a fully offline-capable install for the canonical case.

### Why a single resolution chain (not "current + fallback chain")

`pyenv` exposes a chain (`pyenv versions` returns the currently
sorted list and the active one). `zyenv` ships one resolved version
per call. Selecting fallbacks via "if v0.1.49 is missing, fall back
to v0.1.43" would invite silent version drift, which is exactly the
problem version managers are supposed to prevent. If the pinned
version is missing, the shim refuses to run and tells the user how
to install it.

## Backwards Compatibility

Fully additive. Users who never invoke `zyenv` see no change. The
existing `pip install -e .` system installation continues to work;
once `~/.zyenv/shims` is added to `PATH` in front of the system
Python's `Scripts` directory, `zy` invocations route through the
shim.

If a system `zyenlang` install is present **and** the shim has no
selected version, the user gets the shim's "no version selected"
error, not the system `zy`. Recommend either:

- `zyenv use vX.Y.Z` to point the shim at a managed version, or
- Removing `~/.zyenv/shims` from `PATH` for shells that want the
  system install.

## Reference Implementation

Implementation lives in `zyenlang-v0.1.49` at
`zyenlang/cli/zyenv.py`. The pure-stdlib resolver template is the
`_SHIM_PY` string at the bottom of that module; it is written into
`~/.zyenv/shims/zy_resolver.py` by `zyenv init` so the shim never
needs ZyenLang or any third-party Python package to start a
ZyenLang session.

The `zyenv` console_script is registered in `pyproject.toml` under
`[project.scripts]`:

```toml
zyenv = "zyenlang.cli.zyenv:main"
```

Both `zyenlang` and `zyenlang.cli` are listed in
`[tool.setuptools].packages` so the cli submodule ships in the wheel.

## Open Questions

- **Remote install.** `install-remote vX.Y.Z` from GitHub releases
  or pypi was deliberately deferred — the user is iterating local
  source trees right now and that workflow doesn't need it.
- **`zyenv exec <cmd>`.** Like `pyenv exec`, run a command using the
  resolved version's environment. Useful for one-off scripts.
- **`zyenv rehash`.** Currently the shim is one binary per known
  command name (`zy`, `zyen`). If we add more entry points later,
  this command would re-scan and write new shim files.
- **Telemetry of installed versions.** A `zyenv doctor` would tie
  back into [ZEP-0007](ZEP-0007-zy-doctor.md) — defer to a follow-up.

## References

- [ZEP-0007](ZEP-0007-zy-doctor.md) — `zy doctor` set the precedent
  of CLI tooling living in `zyenlang/cli/`.
- pyenv (https://github.com/pyenv/pyenv), rbenv
  (https://github.com/rbenv/rbenv), nvm
  (https://github.com/nvm-sh/nvm) — direct prior art.
- PEP 517 / PEP 518 — the build-isolation model `--no-build-isolation`
  opts out of.

## Copyright

This document is placed in the public domain.
