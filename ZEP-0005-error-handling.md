# ZEP-0005 — Error Handling Conventions

**English** | [繁體中文](ZEP-0005-error-handling.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0005 |
| **Title** | Error Handling Conventions |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Informational |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

ZyenLang has **no exceptions**. Errors are values: an `int` return
code, a `None` `ptr<T>`, or an empty `List`. This ZEP defines which
shape to use for which kind of error and what the standard library
already does.

## Motivation

When the language has more than one way to signal failure, callers
end up handling some paths and missing others. The compiler can't
help: there are no checked exceptions and no `Result<T, E>` to
exhaustively pattern-match. So the convention has to do the work.

The stdlib already mostly follows the rules below; this ZEP makes
them explicit so new modules don't drift.

## Specification

### Three error shapes

| Shape | When to use it |
|---|---|
| **`int` return code** | The function is an action (write, mkdir, kill). `0` is success, non-zero is the error category. |
| **`ptr<T>` `None`** | The function returns an optional value. `None` = absent or failed; a valid `ptr<T>` = present. |
| **Empty `List`** | The function returns a collection. Empty = no results (whether by failure or by genuinely empty input — that distinction is fine if the operation is idempotent). |

### Picking the right shape

```text
   What does the function return on success?
   │
   ├── nothing meaningful (action)  ──→  int code
   │
   ├── one value of type T          ──→  ptr<T> + None for absent
   │                                     OR (T, int code) split into
   │                                     two functions if the value is
   │                                     a primitive int / str / bool
   │
   └── many values of type T        ──→  List
                                         (caller checks is_empty)
```

The "two functions" split (e.g. `fs.read(path) -> str` + a separate
status check) is the escape hatch when `ptr<str>` would feel heavyweight
for what is morally a string. The stdlib uses this for `fs.read` (returns
`""` on missing file; caller calls `fs.exists` first if they care to
distinguish).

### int code conventions

For functions returning `int`:

- **`0`** — success.
- **Negative** — categorized errors. The stdlib uses small negatives
  (`-1`, `-2`, `-3`) and the function's contract documents what each
  means.
- **Positive** — reserved for downstream tools' exit codes (e.g.,
  `cmd.run` returns the child process exit status verbatim, which is
  positive on most failures).

Example from `std/mem`:

```zy
fn free(p: ptr<Any>) -> int {
    return zl_mem_free(p);
    // Returns: 0=ok, -1=null, -2=not-owned, -3=already-freed
}
```

### `ptr<T>` None conventions

A `None` pointer is the only sanctioned "missing" sentinel for `T`-
shaped values. Callers must guard:

```zy
let p: ptr<Config> = config.load_optional("settings.ini");
if (!mem.is_valid(p)) {
    set p = mem.alloc(default_config());
}
let cfg: Config = *p;
```

`mem.is_valid(p)` is the canonical None check. Do **not** rely on
truthiness of the raw `ptr` — there is no `if (p)` shorthand and the
None sentinel is not necessarily numeric zero in C.

### Panicking is reserved for runtime invariants

Some operations cannot meaningfully return an error:

- `List.get(out_of_range_index)` — there is no fallback value of type
  `Any`.
- Dereferencing a `None` `ptr<T>` after a `mem.is_valid` check would
  have caught it.
- Buffer overflow inside `std/buffer`.

These trigger an immediate process abort with a diagnostic to stderr.
There is no `try` / `catch` to recover. **Do not design new APIs that
panic on bad input**; reserve panic for runtime-detected invariant
violations the caller had no way to prevent.

If a function might be called with bad input from the user, it
returns an `int` code or `None`. If a function is only called from
trusted internal code, it may panic.

## Anti-patterns

### Don't smuggle errors through sentinel values

```zy
// NO — -1 means error, but the caller has no way to know that without
// reading the comment.
fn lookup_age(name: str) -> int {
    // returns -1 if missing
    ...
}

// YES — return ptr<int> so the absent case is explicit.
fn lookup_age(name: str) -> ptr<int> {
    ...
}
```

Exception: when the value's type *itself* excludes valid sentinels
(e.g., a positive-by-construction quantity using `-1` for absent), the
two functions split (`lookup_age` + `has_age`) is cleaner than mixing
ptr and int.

### Don't return `bool` from actions that could give richer information

```zy
// LESS USEFUL
fn write_file(path: str, body: str) -> bool { ... }

// BETTER — int code lets the caller distinguish "no such directory"
// from "permission denied"
fn write_file(path: str, body: str) -> int { ... }
```

### Don't quietly swallow errors

```zy
// NO
fn save(path: str, data: str) -> void {
    fs.write(path, data);   // return code ignored
}

// YES
fn save(path: str, data: str) -> int {
    return fs.write(path, data);
}
```

Even if the immediate caller doesn't care, propagating the code keeps
the option open for the caller's caller.

## Rationale

### Why no exceptions

- ZyenLang lowers to C. Implementing exceptions means either setjmp /
  longjmp (slow, brittle around resource cleanup) or a stack-walking
  unwinder (large runtime).
- Exceptions cross function boundaries invisibly. Callers forget to
  handle them. Static analysis is hard.
- The use cases for exceptions split cleanly into the three shapes
  above. We can write all of the existing stdlib without exceptions
  and the result reads about as cleanly.

### Why no `Result<T, E>`

We'd need to add generic types beyond the existing `ptr<T>` and
`List`, which would mean rewriting the type checker. The two
sanctioned shapes (`ptr<T>` + None, `int` code) handle the same
problems with no new machinery.

### Why distinguish "action" from "optional value"

Pragmatic, not philosophical: the two flow through callers
differently. Actions tend to chain (do-this, then-that); optionals
tend to branch (if present use it, else fallback). Picking the
matching shape keeps the call site readable.

## Reference Implementation

- `std/fs` returns `int` from `write`, `mkdir`, `remove` — actions.
- `std/mem` returns `ptr<T>` from `alloc` and `int` from `free` —
  optional vs action.
- `std/list` returns `int` from `index_of_str` with `-1` for not-found.
  This is the documented exception to "don't smuggle sentinels": the
  return type is a *position*, and `-1` is the universal absent-
  position convention.
- `std/log` returns `int` from `info` / `warn` / `error`.

## Open Questions

- Whether to standardise on an `Error` struct (with a `code: int`,
  `message: str`, `cause: ptr<Error>`) for richer reporting. Out of
  scope for this ZEP; would be a Standards proposal of its own.
- Whether `cmd.run` should expose stderr separately from exit code.
  Currently it returns only the exit code; stderr goes to the
  parent's stderr.

## References

- ZEP-0004 (Global State) — error reporting often interacts with
  shared state (e.g., `errno`-style); we avoid that by keeping codes
  in return values.
- Go's `(T, error)` return convention — ZyenLang's split functions are
  a related shape without the syntactic support.

## Copyright

This document is placed in the public domain.
