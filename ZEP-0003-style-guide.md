# ZEP-0003 — ZyenLang Style Guide

**English** | [繁體中文](ZEP-0003-style-guide.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0003 |
| **Title** | ZyenLang Style Guide |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Informational |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

The everyday style rules for `.zy` source. PEP 8 for ZyenLang, but
shorter because the language surface is smaller.

This ZEP is `Informational`: the compiler does not enforce these
rules. The standard library follows them; user code should too,
but won't be rejected if it doesn't.

## Indentation and spacing

- **4 spaces** per indent level. No tabs in source files.
- One blank line between functions at file scope.
- One blank line between methods inside a `struct`.
- No trailing whitespace.
- Files end with one newline.

## Line layout

- One logical statement per line. ZyenLang's parser allows multi-line
  inside `(...)`, `[...]`, `{...}` but does *not* allow two statements
  on the same physical line.
- Lines should fit in **100 columns**. Break long function calls and
  signatures across lines using the multi-line forms:

  ```zy
  fn long_name(
      first_arg: int,
      second_arg: str,
      third_arg: List
  ) -> int {
      return process(
          first_arg,
          second_arg,
          third_arg
      );
  }
  ```

- Always use braces for `if` / `else` / `for`, even one-line bodies.
  ZyenLang's parser does not reliably accept `if (x) { stmt; }` on a
  single line — always split:

  ```zy
  // YES
  if (n > 0) {
      set n -= 1;
  }

  // NO — parser quirk in v0.1.49
  if (n > 0) { set n -= 1; }
  ```

## Naming

| Kind | Convention | Example |
|---|---|---|
| Function | `snake_case` | `load_buffer`, `classify_word` |
| Local variable | `snake_case` | `cursor_line`, `hi_lines` |
| Function parameter | `snake_case` | `bit_index`, `cursor_col` |
| Struct type | `PascalCase` | `Counter`, `Stack`, `StringMap` |
| Struct field | `snake_case` | `this.base`, `this.list_len` |
| Module name | `lowercase`, ideally one word | `string`, `path`, `fs`, `tk` |
| Module-level constant fn | `snake_case` reads as a noun | `max_pwm()`, `default_port()` |
| Kind / enum-like constant fn | `SCREAMING_SNAKE` allowed | `KIND_KEYWORD()`, `KIND_PLAIN()` |
| File name | `snake_case.zy` | `tk_session_test.zy` |

Module-level "constants" deserve special note: ZyenLang does not allow
top-level `let` / `const` (see [ZEP-0004](ZEP-0004-global-state-and-constants.md)),
so a constant is expressed as a zero-arg function. Name it as the noun
it returns, not as a getter (`max_pwm`, not `get_max_pwm`).

`SCREAMING_SNAKE` is appropriate when the function effectively names an
enum value (`KIND_KEYWORD()`, `MODE_READ()`). For ordinary numeric
constants, prefer plain `snake_case`.

## Type annotations

- **Always** annotate function parameters and return types. The grammar
  requires it.
- **Always** annotate struct fields. The grammar requires it.
- **Recommended** to annotate `let` declarations even when the type is
  obvious: `let n: int = 0;` reads better than `let n = 0;` in a body
  that also touches floats and ptrs.
- Exception: when the right-hand side is a constructor call whose name
  obviously denotes the type, the annotation is noise:

  ```zy
  // Annotate
  let cursor_line: int = 0;
  let visible: bool = true;
  let path: str = "main.zy";

  // Skip — RHS is self-describing
  let xs: List;
  let m: StringMap = map.new();
  ```

## Imports

- One import per line.
- Group in order: standard library first, then user-relative imports.
- Separate the two groups with a blank line.
- **Always** use `as alias`, even when the alias matches the module
  name. This keeps the call sites predictable and prevents collisions
  when imports shift around:

  ```zy
  import <std/string>;          // string.len(s)         OK
  import <std/string> as string; // identical, but explicit — PREFERRED
  import <std/text> as text;
  import <std/fs> as fs;

  import "lex.zy" as lex;
  import "complete.zy" as complete;
  ```

  The IDE's `Ctrl+I` fix-imports walker (`ide_gui.zy`) enforces this
  convention automatically — when in doubt, run it.

## Mutation

- Every mutation requires the `set` keyword. The compiler enforces
  this; the style rule is to **place `set` at the start of the line**
  even inside a `for` header's third clause:

  ```zy
  // PREFERRED
  for (let i: int = 0; i < n; set i += 1) { ... }

  // ALSO ACCEPTED — `i += 1` without `set` works in for-step only
  for (let i: int = 0; i < n; i += 1) { ... }
  ```

  Both compile. The explicit `set` form keeps mutation visually obvious.

## Function shape

- A function does one thing. If the body grows past ~40 lines, extract
  helpers.
- Early-return guards are encouraged:

  ```zy
  fn process(buf: List) -> int {
      if (buf.len() == 0) {
          return 0;
      }
      if (!validate(buf)) {
          return -1;
      }
      // … main path …
  }
  ```

- Avoid deeply nested `if`. Three levels is a code smell; four is a
  bug waiting.

## Structs

- One concept per struct. Resist the urge to bundle.
- Field declarations first, methods after:

  ```zy
  struct Counter {
      let this.base: int;
      let this.step: int;

      fn next() -> int {
          set this.base = this.base + this.step;
          return this.base;
      }
  }
  ```

- A struct that holds only data (no methods) is fine and idiomatic.

## Comments

- Default to **no comment**. A well-named function with a clear body
  should not need one.
- Write a comment when the *why* is non-obvious: a workaround for a
  compiler quirk, a non-trivial invariant, a performance trade-off.
- Don't restate the *what* the code already shows:

  ```zy
  // YES — captures a non-obvious why
  // String concat in a loop is O(N^2); use string.replace which is O(N).
  let normalized: str = string.replace(s, "\r\n", "\n");

  // NO — restates what the code obviously does
  // Set n to 0
  let n: int = 0;
  ```

## What the compiler enforces vs. what this ZEP recommends

| Rule | Source |
|---|---|
| `set` required for mutation | **Compiler** |
| Type annotations on `fn` and `struct` fields | **Compiler** |
| No top-level `let` / `const` | **Compiler**, see [ZEP-0004](ZEP-0004-global-state-and-constants.md) |
| Naming conventions in this ZEP | **Style guide** (not enforced) |
| 4-space indent | **Style guide** |
| 100-col line limit | **Style guide** |
| `as alias` on every import | **Style guide** |

## Open Questions

- Should there be a `zy fmt` tool? Probably yes eventually, but not
  before the surface is more battle-tested.
- Whether to prefer `snake_case` or `SCREAMING_SNAKE` for module-level
  numeric constants — current practice mixes both, the table above is
  guidance, not a hard rule.

## Copyright

This document is placed in the public domain.
