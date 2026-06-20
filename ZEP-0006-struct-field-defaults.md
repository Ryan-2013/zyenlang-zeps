# ZEP-0006 — Struct Field Defaults

**English** | [繁體中文](ZEP-0006-struct-field-defaults.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0006 |
| **Title** | Struct Field Defaults |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

A struct field declaration may carry an optional trailing default
expression. The expression is evaluated at every site where the struct
is constructed; any field whose value is not explicitly supplied by a
struct literal (or any field at all, for declaration-without-init)
takes its declared default. Fields without a declared default keep the
pre-existing C99 zero-fill behaviour.

## Motivation

Before this ZEP, the only way to give a struct field a non-zero default
was to write a constructor function:

```zy
struct Car {
    let this.model: str;
    let this.year: int;
}

fn new_car(model: str) -> Car {
    return Car { model: model, year: 2024 };
}
```

That works, but it forces every caller through the helper and means the
default value is documented in a different place than the field
declaration. The pattern is so common that v0.1.49 already supports the
parallel mechanism for function parameters
(`fn f(a: int, b: int = 1) -> int`). This ZEP extends the same idea to
struct fields, with no new syntax forms — just an optional
`= expression` after the field type.

## Specification

### Syntax

The struct-field grammar is extended from

```text
let this.<name> : <type> ;
```

to

```text
let this.<name> : <type> ( = <expression> )? ;
```

Field declarations without a default are unchanged.

The default `<expression>` is parsed in the surrounding source context
and may use any expression the body of a function would accept, *except*
that it must not reference other fields of the same struct (`this.other`
is not in scope at field-declaration time). Module-level constant
functions, numeric / string / bool literals, and arithmetic are all fine.

### Application order at construction sites

The compiler applies defaults in three construction contexts:

1. **Struct literal with a field list** — `Car { model: "Tesla" }`:
   any field present in the literal wins; any field omitted but with a
   declared default is filled in with the default; remaining fields
   are zero-initialised by C99.

2. **Empty struct literal** — `Car {}`: all defaulted fields are
   filled in; remaining fields are zero-initialised.

3. **Declaration without initialiser** — `let c: Car;` and the
   constructor-form `let c: Car = Car;`: same as the empty struct
   literal.

Within case (1), explicit fields and default-filled fields may be
emitted in any order in the generated C, because C99 designated
initialisers are order-independent.

### Examples

```zy
struct Car {
    let this.model: str;
    let this.year: int = 2024;
    let this.brand: str = "Toyota";
    let this.electric: bool = false;
}

fn main() -> int {
    let c1: Car = Car { model: "Tesla", electric: true };
    // c1 = { model: "Tesla", year: 2024, brand: "Toyota", electric: true }

    let c2: Car = Car { model: "Honda", year: 2020 };
    // c2 = { model: "Honda", year: 2020, brand: "Toyota", electric: false }

    let c3: Car = Car {};
    // c3 = { model: "", year: 2024, brand: "Toyota", electric: false }

    let c4: Car;
    // c4 = { model: "", year: 2024, brand: "Toyota", electric: false }
    return 0;
}
```

### Interaction with declared-field order

Defaults respect the order in which the fields appear in the struct
declaration when both have defaults but neither is overridden by the
literal. The emitted C uses designated initialisers, so order is for
predictability of generated source, not for semantics.

### Forbidden / out of scope

- A default expression **must not** reference `this.<other_field>`.
  Cross-field dependencies would require ordering the field
  initialisers, which this ZEP intentionally avoids.
- A default expression **must not** call into a function that itself
  constructs a struct of the same type (no recursion).
- A default expression **must not** depend on side effects (e.g.,
  global mutable state — there is no global mutable state in ZyenLang
  per [ZEP-0004](ZEP-0004-global-state-and-constants.md), so this is
  automatic).

## Rationale

### Why no ordering constraint

Function parameter defaults must be trailing because positional calls
need an unambiguous fill-in order. Struct literals are keyed by field
name and have no positional form, so any field may carry a default
without affecting the others. This means existing struct declarations
can grow defaults incrementally without rearrangement.

### Why expressions, not just literals

The default goes through `transform_expr`, so it can use module-level
constant functions, simple arithmetic, and string concatenation —
anything an `let` initialiser in a function body would accept. This is
free because the lowering path already exists, and it lets you reuse
declared constants instead of duplicating values.

### Why ban `this.other_field`

To keep the lowering trivial. Cross-field defaults would force the
compiler to topologically sort initialisers and detect cycles. That's
a meaningful amount of complexity for a feature whose use case
(`let this.area: float = this.width * this.height;`) is solved
just as well by a method (`fn area() -> float`).

### Why "Final" on first publication

The feature is small, the semantics are pinned by C99 designated
initialiser behaviour, and the implementation has shipped. Locking the
status to `Final` immediately signals that the syntax is not subject
to bikeshedding; future changes go through a new ZEP that supersedes
this one.

## Backwards Compatibility

Fully backwards compatible.

- Struct field declarations without `= default` parse exactly as
  before.
- Struct literals that provide every field generate the same C as
  before.
- Struct literals that omit fields *and* the struct has no declared
  defaults still produce the same C99 zero-fill behaviour.
- The only new emitted code is the `.field = expr` entries for
  defaulted fields, which is valid C99.

No existing test in `tests/` had to be modified; all 14 `text_test`
cases still pass and `examples/add.zy` is unaffected.

## Reference Implementation

- `zyenlang/transpiler.py`:
  - `StructDef` gains a `defaults: Dict[str, str]` field.
  - The field regex in `collect_signatures()` accepts an optional
    trailing `= <expression>`.
  - The mirror regex in the emit pass accepts the same form.
  - `convert_struct_literal()` fills in unspecified fields using the
    struct's declared defaults.
  - New helper `_struct_default_init(struct_name, ctx)` is used by
    `parse_var_decl()` for both the declaration-without-init form
    (`let c: Car;`) and the constructor form (`let c: Car = Car;`).

## Open Questions

- Whether a default-only `Car {...}` (literal with no fields explicit
  but inside the braces) should be a stylistic recommendation over
  `let c: Car;`. The two produce identical C; this ZEP treats them as
  interchangeable.
- Whether to extend the same syntax to function parameters' missing
  `*args` / `**kwargs` (it cannot — see ZEP / memory note rejecting
  variadic kwargs).

## References

- [ZEP-0004](ZEP-0004-global-state-and-constants.md) — the structural
  reason struct singletons exist; this ZEP makes them ergonomic.
- v0.1.49 function default parameters (in the main repo README) — the
  precedent this ZEP mirrors.
- C99 designated initialisers, §6.7.8 — the underlying C feature that
  makes zero-fill of unspecified fields free.

## Copyright

This document is placed in the public domain.
