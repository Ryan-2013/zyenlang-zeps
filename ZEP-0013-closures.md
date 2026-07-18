# ZEP-0013 — Closures and Lambda Lifting

**English** | [繁體中文](ZEP-0013-closures.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0013 |
| **Title** | Closures and Lambda Lifting |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Standards |
| **Created** | 2026-06-21 |
| **Post-History** | 2026-06-21 |
| **Supersedes** | parts of [ZEP-0010](ZEP-0010-first-class-functions.md) (banned nested fn) |

## Abstract

ZyenLang allows a `fn` definition nested inside another `fn` body. The
nested fn captures outer-scope identifiers by value at construction
time and is returned (or stored) as a `fn(...)->T` value. The compiler
lambda-lifts the nested fn to file scope and packages captured state
into a heap-allocated environment record. Captures are **read-only**
inside the closure body.

## Motivation

[ZEP-0010](ZEP-0010-first-class-functions.md) shipped first-class fn
values but explicitly banned nested fn definitions, on the grounds
that closures need heap-allocated environments and lifetime tracking.
The user accepted that for one round and immediately hit the obvious
pattern that ZEP-0010 could not express:

```zy
fn make_repeater(f: fn(int,int,int)->int, rounds: int) -> fn(int)->int {
    fn back_fn(k: int) -> int {
        for (let i = 0; i <= rounds; set i += 1) {
            f(i, i, k);
        }
        return 0;
    }
    return back_fn;
}
```

The struct-field workaround (`Repeater { f, rounds }` with a `.run(k)`
method) works but pushes the boilerplate onto the *caller* instead of
the factory. For factories that produce many small closures the
struct-per-closure pattern is verbose.

This ZEP adds the missing direct form by giving up the original "no
heap allocation" stance, while keeping the cost contained: closures
are the *only* heap-allocating fn value, and the cost is one `malloc`
per closure construction.

## Specification

### Syntax

Nested `fn` definitions are now legal inside another `fn` body:

```zy
fn outer(<params>) -> <ret> {
    ...
    fn inner(<params>) -> <ret> {
        // body — may reference outer's params and locals declared
        // earlier in `outer`'s body
    }
    ...
    return inner;       // or `let x = inner;` etc.
}
```

The nested fn's name is a local fat-value symbol of the outer fn (not
callable by its bare name from anywhere else). Inside the outer fn's
body, after the nested fn's definition line, the name is in scope as
a `fn(<param-types>)-><ret>` value.

### Capture analysis

The compiler computes the set of *free variables* of the nested fn:
identifiers used inside its body that

- are not declared as one of its parameters,
- are not declared by `let` / `for (let ...)` inside its body,
- are not file-scope (top-level fns, structs, consts),
- are not ZyenLang keywords or built-in type names,
- **are** in the enclosing fn's scope at the nested fn's definition
  line (its params + earlier-declared locals).

The matching set becomes the closure's captures.

### Capture semantics

Captures are **read-only snapshots** taken at the closure-construction
moment. Concretely:

1. At the nested fn's definition line, the compiler emits a `malloc`
   for an env struct holding one field per capture, then copies each
   captured local's *current* value into the env, then builds the fat
   value `ZL_Function { call, env, owner, signature }`.
2. Each call to the closure copies env's captures back into local
   variables of the lifted fn (so the body can reference them by
   their original name).
3. Assigning to a capture inside the closure body (`set <capture> = ...`)
   mutates the per-call local copy. Neither the env nor the original
   outer-scope binding is affected.
4. Subsequent calls re-read the captures from env: they always see the
   construction-time snapshot.

If mutable state per closure is needed, use a struct with a fn-typed
field ([ZEP-0010](ZEP-0010-first-class-functions.md) +
[ZEP-0011](ZEP-0011-struct-body-methods.md) pattern) — the struct
itself holds the mutable state.

### Lifetime

The env record is managed by atomic ARC as standardized by
[ZEP-0014](ZEP-0014-managed-pointers-and-arc.md). Copying, assigning,
capturing, storing, or returning a closure retains its owner. Scope exit and
replacement release it. The final release runs an environment destructor that
recursively releases managed captures.

### Forbidden constructs (still)

- **A nested fn that captures a `List`** — captures are by-value; a
  `List` capture would copy the struct but share the heap buffer,
  leading to confusing aliasing. The compiler currently does not
  reject this at parse time but the behaviour is implementation
  defined. Avoid.
- **Two-level nesting** — `fn outer() { fn middle() { fn inner() {...} } }`
  is undefined; the implementation does not walk past one level.
- **Mutating a captured fn value** — same as above; treat captures as
  read-only.
- **Lambda literals** — there is still no `fn(x) {...}` anonymous
  expression. Define a named nested fn and reference it by name.

## Rationale

### Why lambda lifting (instead of fat raw closures)

Lambda lifting moves every nested fn to file scope as a regular C
function. The closure value contains a lifted call pointer, environment,
ARC owner, and canonical signature. The
alternative (fat closure with embedded trampoline) requires runtime
code generation or libffi, neither of which fits the project. Lambda
lifting is the textbook compile-target-to-C technique.

### Why read-only captures

True read-write captures need shared cells (Python's `cell` type,
C++'s `std::shared_ptr<int>`, etc.) so multiple closures and the
outer scope can all see one mutation. ZyenLang's ARC manages the lifetime of
each read-only snapshot, but it does not change capture mutation semantics:
every closure still has its own snapshot.

The struct-field pattern already handles the mutable-shared case
cleanly (the struct is the shared cell), so we are not losing
expressiveness.

### Why automatic ARC

Function values move through locals, struct fields, captures, assignments, and
returns. Compiler-inserted ARC gives those paths one ownership rule without
adding retain/release syntax to ZyenLang source. Atomic control blocks also let
C callback adapters safely retain and release closure ownership across threads;
the captured data itself does not automatically become thread-safe.

## Backwards Compatibility

- All ZEP-0010 fn-value programs still compile. Named-fn references
  now flow through a generated thunk + const fat literal (`<name>_zlfnval`)
  with no environment owner. The thunk discards the env argument and forwards
  to the original named fn.
- Call sites that used `f(args)` on a fn-typed local now compile to
  a signature-checked ABI helper in C. The ZyenLang surface syntax does not
  change.
- The nested-fn error from ZEP-0010 no longer fires — the same source
  text now compiles to a closure.

## Reference Implementation

The current reference implementation lives in ZyenLang v0.1.53
(`zyenlang/transpiler.py`):

### New / changed pieces

| Piece | Purpose |
|---|---|
| `emit_fn_typedefs` | Emit signature-specific checked-call helpers over the common `ZL_Function` ABI |
| `emit_fn_thunks` (new) | Per top-level fn: `<name>_zlthunk` + `<name>_zlfnval` |
| `coerce_named_fn_to_fnval` (new) | Substitute bare `<name>` → `<name>_zlfnval` when expected type is fn |
| `transform_function_calls` | Route calls on fn-typed values through the signature-specific checked helper |
| `transform_method_calls` | Apply the same checked call path to fn-typed struct fields |
| `scan_nested_fns` (new pre-pass) | Walk fn bodies; for each nested fn, compute captures + register lifted fn in ctx |
| `emit_lifted_env_structs` (new) | One `typedef struct ZL_env_<outer>_<inner> { … };` per closure |
| `emit_lifted_fn_prototypes` (new) | Forward-declare each `<outer>_<inner>_zllifted` |
| `emit_lifted_fn_bodies` (new) | Define each lifted fn at end of file; hoist captures into locals then transpile body normally |
| `emit_function_body` | On nested fn definition: allocate an ARC env, retain managed captures, and build a `ZL_Function` local |

### Codegen order

1. `scan_nested_fns` — registers lifted fn typedefs.
2. `collect_fn_typedefs` — picks up all fn types.
3. `emit_struct_forward_decls`.
4. `emit_fn_typedefs` — signature-specific checked-call helpers.
5. `emit_struct_defs`.
6. `emit_lifted_env_structs`.
7. `emit_function_prototypes` — real fns.
8. `emit_lifted_fn_prototypes`.
9. `emit_fn_thunks` — named-fn thunks + fnval constants.
10. (runtime helpers, std module C stubs, etc.)
11. Main fn emit loop.
12. `emit_lifted_fn_bodies` — lifted closure bodies last.

### Capture analysis cheat sheet

- `inner_local_bindings` = inner params + identifiers that appear after
  `let `, `const `, or `for (let ` inside the body.
- `used` = all identifiers in the body (string literals stripped).
- `captures` = `used - inner_local_bindings - keywords - file_scope`,
  filtered to those present in the outer fn's scope-at-definition.

### Lifted fn body skeleton

```c
static int <outer>_<inner>_zllifted(void* env_v, int k) {
    ZL_env_<outer>_<inner>* __zl_env = (ZL_env_<outer>_<inner>*)env_v;
    int rounds = __zl_env->rounds;            // hoisted capture
    ZL_Function f = __zl_env->f;               // borrowed snapshot during call
    /* original nested-fn body, transpiled as-is */
}
```

Tests at `tests/closure_test.zy`, `tests/fn_chain_test.zy`, and
`tests/c_module_callback_test.zy` cover capture semantics, ARC ownership,
immediate chained calls, and callbacks retained by C compatibility code.

## Open Questions

- **Mutable captures via cell.** Adding a `cell<T>` wrapper that
  closures share (Python-style) would restore mutation across calls.
  Out of scope for the current reference implementation.
- **Closure-in-closure.** Two-level nesting needs the lifted middle
  fn's body to itself perform lambda lifting. Implementation is
  recursive in `scan_nested_fns` but currently not exercised; treat
  as undefined.
- **List captures.** Either reject at parse time or document that the
  captured `ZL_List` is a shallow copy.
- **Method-reference captures.** `let f = some_struct.method;` is
  still unsupported (ZEP-0010 open question). Closures don't change
  that.

## References

- [ZEP-0003](ZEP-0003-style-guide.md) — surface-syntax discipline (no
  lambda literal, no arrow operators).
- [ZEP-0010](ZEP-0010-first-class-functions.md) — fn-value foundation
  this ZEP builds on. The "no nested fn" rule from ZEP-0010 is
  superseded; everything else still holds.
- [ZEP-0011](ZEP-0011-struct-body-methods.md) — struct-method form,
  the recommended pattern when mutable shared state is required.

## Copyright

This document is placed in the public domain.
