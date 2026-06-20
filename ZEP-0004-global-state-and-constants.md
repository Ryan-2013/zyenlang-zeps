# ZEP-0004 — Global State and Module-Level Constants

**English** | [繁體中文](ZEP-0004-global-state-and-constants.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0004 |
| **Title** | Global State and Module-Level Constants |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

ZyenLang **rejects top-level `let` and `const` declarations**. A
program's only file-scope constructs are `import`, `struct`, and `fn`.
This ZEP explains the rule, why it exists, and the four idioms that
replace what other languages use globals for: module-level constants,
read-only configuration, mutable shared state, and persistent state.

## Motivation

Earlier ZyenLang versions silently dropped top-level `let` declarations:
the parser accepted them, but they never emitted to C, and code that
referenced them failed at gcc with `undeclared identifier`. The
behaviour was a P2 bug for users and an attractive nuisance for anyone
porting code from Python or JavaScript.

v0.1.49 closes the hole on both sides: the compiler **rejects** top-
level `let` / `const` with a clear error message, and this ZEP codifies
the four sanctioned alternatives.

## Specification

### The rule

The only constructs allowed at file scope are:

```text
import <std/...>;
import "..." as ...;
struct Name { ... }
fn name(...) -> T { ... }
```

These are rejected at file scope and produce a compile-time error:

```zy
let g: int = 1;       // ERROR
const max_pwm = 1000; // ERROR
set g = 2;            // ERROR — no `set` outside of function body anyway
```

Mutable variables only exist inside `fn` bodies. Constants are
expressed as zero-arg `fn`. Shared state is passed explicitly or
encapsulated in a struct or on disk.

### Pattern 1 — Module-level constant

For a numeric or string value that is the same everywhere in the program,
expose a **zero-argument function**:

```zy
fn max_pwm() -> int {
    return 1000;
}

fn default_port() -> int {
    return 8080;
}

fn app_name() -> str {
    return "ZyenLang IDE";
}
```

Use sites read like normal calls:

```zy
let pwm: int = clamp(target, 0, max_pwm());
```

This is the *primary* idiom and the one to reach for first.

#### Sub-pattern: enum-like constants

When a group of constants forms an enum, name them with a shared
prefix:

```zy
fn KIND_PLAIN()   -> int { return 0; }
fn KIND_KEYWORD() -> int { return 1; }
fn KIND_TYPE()    -> int { return 2; }
fn KIND_NUMBER()  -> int { return 3; }
```

This is exactly how `std/text/lex.zy` (the IDE's syntax highlighter)
defines its `KIND_*` codes. Naming convention is covered in
[ZEP-0003](ZEP-0003-style-guide.md).

### Pattern 2 — Read-only configuration from disk

For values that change per environment (paths, ports, feature flags),
load them once from a config file at startup using `std/config`:

```zy
import <std/config>;

fn load_settings() -> StringMap {
    return config.load("settings.ini");
}

fn main() -> int {
    let cfg: StringMap = load_settings();
    let port: int = string.to_int(cfg.get("port", "8080"));
    serve(port);
    return 0;
}
```

The caller passes `cfg` (or a derived `port`) down explicitly. No file-
scope state needed.

### Pattern 3 — Mutable shared state via a struct singleton

When several functions truly need to share mutable state, declare a
struct, instantiate it once in `main`, and **pass it by reference** (or
store it in a `List` cell) to every function that needs it:

```zy
struct AppState {
    let this.cursor_line: int;
    let this.cursor_col: int;
    let this.dirty: bool;

    fn mark_dirty() -> void {
        set this.dirty = true;
    }
}

fn handle_keystroke(state: AppState, key: str) -> void {
    if (string.eq(key, "Right")) {
        set state.cursor_col += 1;
        state.mark_dirty();
    }
}

fn main() -> int {
    let state: AppState = AppState { cursor_line: 0, cursor_col: 0, dirty: false };
    handle_keystroke(state, "Right");
    return 0;
}
```

This is more verbose than a global, but the data flow is visible. The
ZyenLang IDE (`apps/zyide_gui.zy`, ~2500 lines) uses exactly this
pattern.

### Pattern 4 — Persistent state across function calls via disk

When state must survive across calls *and* across program runs, write
it to a file. `std/log` does this:

```zy
import <std/fs>;

fn log_path() -> str {
    return ".zy_log_file";
}

fn write_log(message: str) -> int {
    return fs.append(log_path(), message + "\n");
}

fn read_log() -> str {
    return fs.read(log_path());
}
```

The "global" here is the file on disk, not a process-memory variable.
Pros: survives crashes, observable from outside the program, no
synchronization. Cons: I/O cost; only suitable for low-frequency state.

## Rationale

### Why ban top-level `let` outright

Three reasons:

1. **The old behaviour was a foot-gun.** Silent drop + gcc error is
   the worst possible outcome — beginner-confusing and hard to
   diagnose.
2. **C interop is cleaner.** Every `.zy` function maps to a C
   function; every `struct` to a C struct; every `import` to a
   prepended source file. There is no clean lowering for top-level
   `let` without inventing a Zyen-side `static` initialization order
   that mirrors C's (which famously bites everyone who writes a
   shared library).
3. **The four patterns above cover every real use case.** We have
   2500-line apps and 40-module stdlibs that follow these rules.

### Why zero-arg `fn` instead of a `const` keyword

We *could* have added file-scope `const fn` or `const name = value;`.
Reasons we didn't:

- ZyenLang already emits C prototypes for every `fn`. Treating
  constants as zero-arg `fn` is free — no new lowering path.
- The call-site `max_pwm()` reads as a value; the parens are minor noise.
- Constants don't bind early. If the value should change for a future
  platform, the function body is the natural place to branch.

### Why not allow file-scope `struct singleton = StructName { ... };`

Same as #2 above: it would require static initialization ordering
machinery we don't have and don't need. Instantiate singletons in
`main` and pass them down. The data flow is obvious; the price is one
extra parameter per call.

## Backwards Compatibility

No breaking change for v0.1.49+. The behaviour pre-v0.1.49 was the
silent-drop bug; the v0.1.49 compiler now raises a clear error
instead. Any user code that *did* rely on the silent drop was broken
already and the new error is strictly better than the old gcc-
undeclared message.

## Reference Implementation

- Compiler: `zyenlang/transpiler.py` — top-level statement classifier
  rejects `let` / `const` and emits the diagnostic.
- Patterns 1 & 2 in the wild: `apps/zyide_gui.zy` uses zero-arg `fn`
  for theme colors (`col_bg()`, `col_panel()`, …) and dimensions
  (`win_w()`, `header_h()`, `body_y0()`, …).
- Pattern 3 in the wild: `apps/zyide_gui.zy` passes ~25 state values
  through every `redraw(...)` call. Verbose, yes; explicit, yes.
- Pattern 4 in the wild: `zyenlang/std/log.zy` stores logger
  configuration on disk between function calls.

## Open Questions

- Whether to add a `const` keyword *inside* `fn` bodies (currently
  there is `const x: T = v;` but it's effectively `let` + no `set`
  allowed). Out of scope for this ZEP.
- Whether `import "..." as alias;` should bring `struct` definitions
  in qualified (`alias.Type`) instead of global (`Type`). Currently
  imported user structs are global and the alias is stripped from type
  positions — see the `_strip_module_prefix_type` helper. A future ZEP
  may revisit.

## References

- ZEP-0003 (Style Guide) — naming for the constant-fn pattern.
- C: `static` storage class and initialization order issues — this ZEP
  ducks both.
- Python: module-level `_CONSTANT` convention — ZyenLang's `max_pwm()`
  is the same idea, expressed as a call.

## Copyright

This document is placed in the public domain.
