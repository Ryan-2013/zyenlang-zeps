# ZEP-0011 — Struct-Body Method Definitions

**English** | [繁體中文](ZEP-0011-struct-body-methods.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0011 |
| **Title** | Struct-Body Method Definitions |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-06-21 |
| **Post-History** | 2026-06-21 |

## Abstract

ZyenLang struct methods are defined **inside** the struct body, after
the field declarations. There is no external `fn StructName.method(...)`
form. The body of every method has an implicit `this` of pointer-to-
struct type. This ZEP fixes the rule that was previously implicit in
v0.1.49.

## Motivation

A `struct` declaration should be self-describing: a reader who scans
just the struct body must see every field **and** every method the type
exposes. Splitting a method's signature off into a separate `fn
Repeater.run(...)` block somewhere else in the file fragments the
type, and lets a method be added that the struct definition itself
never advertised. The C tradition of "data here, ops far away" is a
known ergonomic loss; ZyenLang collapses the two back into one block.

## Specification

### Syntax

```text
struct <Name> {
    let this.<field>: <type>;
    let this.<field>: <type> = <default>;     // ZEP-0006
    ...
    fn <method>( <params> ) -> <ret> {
        ...
        return <expr>;
    }
    fn <method>( <params> ) -> <ret> {
        ...
    }
}
```

- Field declarations precede method definitions by convention; the
  parser accepts them in any order, but mixing reduces readability.
- Method headers follow the same grammar as top-level functions
  (`fn name(<params>) -> <ret>`), including [ZEP-0010](ZEP-0010-first-class-functions.md)
  fn-typed parameters and return types.
- Methods may be omitted entirely; a struct with only field
  declarations is still valid.
- `fn StructName.method(...)` written **outside** the struct body is
  a syntax error. There is no external form.

### Implicit `this`

Inside a method body, the identifier `this` is a pointer to the
receiving struct instance. It is *not* listed in the method header.
The C-side function is named `<Struct>_<method>` and takes
`<Struct>* this` as its first parameter.

```zy
struct Counter {
    let this.value: int;
    fn bump(by: int) -> int {
        set this.value += by;
        return this.value;
    }
}
```

compiles to:

```c
int Counter_bump(Counter* this, int by) {
    this->value += by;
    return this->value;
}
```

### Calling methods

Through a value or a pointer, syntax is the same:

```zy
let c: Counter = Counter { value: 0 };
c.bump(5);
```

The transpiler emits `Counter_bump(&c, 5)`. For `ptrstruct<Counter>` it
emits `Counter_bump(c, 5)` directly.

A method may call another method on the same struct through `this`:

```zy
struct Stats {
    let this.pass: int;
    let this.fail: int;
    fn check(cond: bool) -> int {
        if (cond) { set this.pass += 1; } else { set this.fail += 1; }
        return 0;
    }
    fn total() -> int {
        return this.pass + this.fail;
    }
}
```

### Forbidden constructs

- **External `fn Struct.method(...)`** — parse error. Move the method
  into the struct body.
- **Method definitions that omit `fn`** — every method begins with the
  literal `fn` keyword.
- **`this` rebinding** — `set this = ...` is rejected. `this` is a
  read-only pointer.

## Rationale

### Why inline, not external

A struct definition is the canonical place a user looks to learn what
the type can do. If methods can be defined anywhere in the file, the
struct definition is an incomplete contract. The external form also
invites the C++ "header vs. implementation" split that ZyenLang
explicitly does not want.

### Why no `impl` block

Rust uses `impl StructName { fn ... }` to add methods after the struct
definition. That gives flexibility (methods can be added in another
module, methods can be defined per trait) but the cost is the same
indirection ZyenLang is rejecting. v0.1.49 has no traits and no
cross-module method addition, so an `impl` block would only ever
restate what is already in the struct body. Skipping it keeps the
grammar smaller.

### Why `this` is not a parameter

Listing `this` in every method header is C boilerplate. ZyenLang already
knows the receiver type (it's the enclosing struct) and the pointer
nature is a fixed convention. Removing it from the header makes the
method signature read like the user's mental model of the operation.

## Backwards Compatibility

Fully backwards compatible — this ZEP codifies what v0.1.49 already
implements. Existing programs that defined methods inline continue to
work unchanged. Any program that *tried* to use the external form would
have already failed with `top-level statement is not allowed`.

## Reference Implementation

Implementation lives in `zyenlang-v0.1.49`
(`zyenlang/transpiler.py`):

- `collect_signatures` at the struct-body branch registers each
  `fn <name>(...)` inside a struct body as a top-level function named
  `<Struct>_<name>` with a `this: ptrstruct<Struct>` first parameter.
- The emit pass mirrors the same loop, emitting
  `c_function_signature(<Struct>_<name>, ret_type, c_params) + " {"`
  and dispatching the body to `emit_function_body`.
- `transform_method_calls` rewrites `obj.method(args)` to
  `Struct_method(&obj, args)`, or `Struct_method(obj, args)` when `obj`
  is already a `ptrstruct<Struct>`.

There is no separate parser pass — methods are parsed in the same
walk as the struct.

## Open Questions

- Whether to allow `static` (associated) methods that have no `this`
  receiver. Today every struct-body `fn` gets a `this` parameter; a
  static method would skip it. Defer until a real use case appears.
- Whether to allow method prototypes (`fn name(...) -> T;` with no
  body) inside the struct body for "declared here, defined elsewhere".
  Currently rejected; revisit if cross-file method addition lands.

## References

- [ZEP-0006](ZEP-0006-struct-field-defaults.md) — struct field default
  initialisers, which use the same struct-body parse pass.
- [ZEP-0010](ZEP-0010-first-class-functions.md) — fn-typed fields,
  which a struct-body method commonly reads through `this.field(args)`.

## Copyright

This document is placed in the public domain.
