# ZEP-0010 — First-Class Function Values

**English** | [繁體中文](ZEP-0010-first-class-functions.zh-TW.md)

> **Note (2026-06-21):** the "nested function definitions are
> forbidden" rule has been **superseded by [ZEP-0013](ZEP-0013-closures.md)**,
> which allows nested `fn` and lambda-lifts them into closures with
> heap-allocated, read-only captures. Everything else in this ZEP
> still holds. Since ABI v2, fn values use
> `ZL_Function { call, env, owner, signature }`. A bare named fn has no
> environment owner; closures retain an ARC-managed environment. See
> [ZEP-0014](ZEP-0014-managed-pointers-and-arc.md).

| Field | Value |
|---|---|
| **ZEP** | 0010 |
| **Title** | First-Class Function Values |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Standards |
| **Created** | 2026-06-21 |
| **Post-History** | 2026-06-21 |

## Abstract

ZyenLang gains a `fn(<types>)-><type>` type expression that turns a
named top-level function into a first-class value. A `fn`-typed value
can be stored in a variable, passed as a parameter, returned from a
function, and held in a struct field. It cannot be synthesised at
runtime: there is no lambda, no closure, no captured environment. The
only legal source of a `fn`-typed value is a reference to an
already-defined top-level function (or a `fn`-typed value derived from
one).

## Motivation

The user repeatedly hits patterns that need "pass the operation, not
the result" — comparators, reducers, dispatch tables, strategy
selection. Before this ZEP, the only options were:

1. Hard-coded `if/else` on a string or enum identifier inside the
   callee.
2. Generating one specialised function per case.

Both inflate the codebase and make extension awkward. Mainstream
languages solve this with function pointers (C), function references
(C++ `std::function`), or full closures (JavaScript, Python). The user
explicitly rejected closures and lambda syntax as too complex (see
[ZEP-0003](ZEP-0003-style-guide.md) and the
`zyenlang-syntax-no-complex` memory note). C-style function pointers
sit cleanly in the middle: pass the address of a *named* function;
nothing more.

## Specification

### Type syntax

```text
fn ( <param-type> ( , <param-type> )* ) -> <return-type>
fn ( ) -> <return-type>
fn ( <param-type> ( , <param-type> )* )            # implicit -> void
fn ( )                                              # implicit -> void
```

- Parameter types are listed by type only. **Parameter names are
  forbidden inside the type**: write `fn(int,int)->int`, not
  `fn(a:int,b:int)->int`. Names belong on the declaration of the
  underlying function, not on its type.
- The return type may be any normal ZyenLang type, including another
  `fn(...)->T`. v0.1.49 supports one level of nesting; arbitrarily
  deeply nested fn types in *param* position are out of scope for this
  ZEP.
- Whitespace is allowed everywhere a normal type accepts it.

### Where the type is allowed

A `fn(...)->T` type expression may appear anywhere a built-in type can:

```zy
// As a parameter type
fn apply(op: fn(int,int)->int, x: int, y: int) -> int { ... }

// As a return type
fn pick(name: str) -> fn(int,int)->int { ... }

// As a local variable type
let f: fn(int,int)->int = add;
let g: fn(int)->int;             // defaults to NULL until assigned

// As a struct field type
struct Reducer {
    let this.op: fn(int,int)->int;
    let this.seed: int;
}
```

### Where the value comes from

The only legal RHS for a `fn`-typed assignment, parameter, or field is:

1. A bare reference to a named top-level function: `add`, `sub`,
   `MyStruct.method` (the last reserved for a possible future
   extension — not allowed in v0.1.49).
2. A function-call expression whose return type is a `fn(...)->T`.
3. The literal `NULL` is **not** a writable value in source; a
   declaration without an initialiser implicitly starts at NULL.

A `fn`-typed variable that is NULL must be reassigned before being
called. Calling a NULL `fn` value is undefined (the C backend will
crash).

### Calling a `fn`-typed value

Calls use the same syntax as a direct function call:

```zy
let f: fn(int,int)->int = pick("add");
let r: int = f(3, 4);             // emits C `f(3, 4)`
```

Through a struct field:

```zy
fn Reducer.run(values: List) -> int {
    set acc = this.op(acc, v);    // emits C `this->op(acc, v)`
}
```

### Type checking

Two `fn` types are equal iff their parameter lists and return types are
equal after whitespace normalisation. A `fn(int,int)->int` value is
not assignable to a `fn(int,int)->float` variable.

Arguments at the call site are not coerced beyond the existing rules
for direct calls — the C compiler enforces the function-pointer
signature, so a mismatch surfaces as a backend error.

### Forbidden constructs

- **Nested function definitions** — `fn outer() { fn inner() {...} }`
  is a compile-time error. The compiler emits:

  ```text
  line N: nested function definitions are not allowed; define `fn` at
  file scope and use a struct field of fn type to bind state, e.g.
  `let this.f: fn(int)->int;`
  ```

  This rule is the load-bearing one. It guarantees no captured
  environment ever has to exist, so the runtime stays equivalent to a
  C function-pointer table.

- **Parameter names inside a `fn` type** — see the syntax section.

- **Anonymous function literals** — there is no `fn(x){...}`
  expression syntax in v0.1.49 and this ZEP does not propose one.

## Rationale

### Why function pointers and not closures

Closures capture the surrounding scope. To compile closures to C the
compiler must (a) decide which outer variables to capture, (b)
heap-allocate an environment record, (c) bundle it with the function
pointer into a fat value, (d) free the environment when the closure
is no longer reachable. Every step pushes ZyenLang closer to needing
a garbage collector or a borrow checker. Both are out of scope for
v0.1.49 and would block the "C-like surface" goal.

Named top-level functions have none of those problems. They are
already compiled to ordinary C functions; their addresses are valid
for the lifetime of the program; there is nothing to allocate.

### Why ban parameter names in the type

`fn(a:int,b:int)->int` and `fn(int,int)->int` are the same type — the
names are only relevant to the function's *definition*. Allowing them
in type expressions invites users to write three names for the same
arguments (definition, declaration, type) that can drift out of sync.
Banning them at parse time keeps one source of truth.

### Why ban nested `fn`

Once you allow `fn` inside another `fn`, the next obvious step is to
let the inner one reference the outer's locals. That is a closure.
The ban makes the rule mechanical: every `fn` lives at file scope,
which is the same rule that already exists for top-level functions
today. No new scope concept is introduced.

### Why a struct field is the closure replacement

The classic "I need to bundle state with a function" pattern becomes:

```zy
struct Reducer {
    let this.op: fn(int,int)->int;
    let this.seed: int;
}
```

The user *writes the environment out* as named fields, instead of the
compiler inferring it from variable usage. This is the same trade-off
as Go's "method on a struct" pattern vs. JavaScript's free closures.
The struct version is verbose but self-documenting and has zero
implicit allocation.

## Backwards Compatibility

Fully backwards compatible.

- The token sequence `fn(` only had meaning at the start of a top-level
  declaration or struct method. Existing code does not put `fn(` after
  `:` or `->`, so no existing source becomes ambiguous.
- Existing struct field defaults ([ZEP-0006](ZEP-0006-struct-field-defaults.md))
  still apply; a `fn`-typed field without a default initialiser starts
  at NULL (C99 zero-fill), which is the safe sentinel.

## Reference Implementation

The current reference implementation lives in ZyenLang v0.1.53
(`zyenlang/transpiler.py`):

- **Helpers** — `is_fn_type`, `parse_fn_type`, `fn_typedef_name`,
  `register_fn_typedef`, `emit_fn_typedefs`, `collect_fn_typedefs`,
  `parse_fn_header_line`.
- **Type grammar** — `TYPE_RX` module-level regex constant now
  accepts both simple and fn types; reused in struct-field,
  `let`-decl, `parse_var_decl`, and the struct member emit pass.
- **`c_type`** maps `fn(...)->T` to the deterministic typedef name
  `ZL_fn_<params>_to_<ret>`.
- **`validate_user_type`** recurses into fn-type pieces and rejects
  parameter names.
- **`infer_type`** returns the fn type for a bare named-function
  reference, and resolves call-through-fn-typed-var expressions.
- **`transform_method_calls`** detects fn-typed struct fields and
  emits `obj.field(args)` / `obj->field(args)` instead of the usual
  `Struct_method(&obj, args)` dispatch.
- **`emit_function_body`** raises an error on any nested `fn `.
- **Codegen ordering** — struct forward declarations are emitted
  first, then fn typedefs (which may reference struct names), then
  full struct bodies (which may contain fn-typed fields using the
  typedefs), then function prototypes.

Tests at `tests/fn_value_test.zy` cover all four use cases
(parameter, return, struct field, uninitialised-then-set).

## Open Questions

- Whether to allow a future shorthand for the common case of a
  zero-arg, void-return `fn` (callbacks). Currently you must write
  `fn()->void`; some languages allow simply `fn()`. This ZEP already
  accepts the omitted `->void`, but the user-facing style guide will
  pick a canonical form.
- Whether to extend `fn` types into nested positions arbitrarily
  (`fn(fn(int)->int)->fn(int)->int`). Implementation effort is small
  (deeper regex / parser changes); the question is whether the
  resulting code stays readable.
- Method references — `let m = some_struct.method;` is not yet
  supported; struct methods always go through the implicit `this`. A
  future ZEP may add bound-method values.

## References

- [ZEP-0003](ZEP-0003-style-guide.md) — surface-syntax discipline.
- [ZEP-0004](ZEP-0004-global-state-and-constants.md) — struct
  singletons as the existing "bundle state without globals" pattern;
  fn-typed fields generalise that pattern.
- C99 §6.7.5.3 (Function declarators) and §6.5.2.2 (Function calls) —
  the C semantics this ZEP compiles to.

## Copyright

This document is placed in the public domain.
