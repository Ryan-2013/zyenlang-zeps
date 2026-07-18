# ZEP-0014 - Managed Pointers and Automatic Reference Counting

**English** | [Traditional Chinese](ZEP-0014-managed-pointers-and-arc.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0014 |
| **Title** | Managed Pointers and Automatic Reference Counting |
| **Author** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-07-18 |
| **Requires** | [ZEP-0010](ZEP-0010-first-class-functions.md), [ZEP-0013](ZEP-0013-closures.md) |

## Abstract

ZyenLang keeps explicit pointer syntax while representing language-level
`ptr<T>` values as checked managed handles. Owned pointers and closure
environments use compiler-inserted atomic reference counting (ARC). Borrowed
pointers remain possible for C interoperability and local address-taking.

This ZEP specifies recursive pointer types, owned allocation, dereference,
casts, aliasing, return-value ownership transfer, managed struct fields, and
the limits of the safety model.

## Pointer kinds

### Ordinary local storage

```zy
let value: int = 10;
```

An ordinary local has automatic storage duration. Declaring it does not imply
a heap allocation. Its lifetime ends when its function scope exits.

### Borrowed pointer

```zy
let value: int = 10;
let borrowed: ptr<int> = &value;
```

`&value` creates a `ptr<int>` whose ARC owner is null. It does not extend the
lifetime of `value`. Returning an obvious pointer to stack storage is rejected,
but ZyenLang does not provide a complete borrow checker.

### Owned pointer

```zy
let *owned: ptr<int> = 10;
```

`let *` allocates an owned cell, initializes it with the right-hand value, and
stores a managed `ptr<int>` handle in `owned`. Copies retain the same owner;
scope exit releases one reference.

## Recursive pointer types

`ptr<T>` is recursive. The pointee may itself be another pointer:

```zy
let *two: ptr<ptr<int>> = 10;
let *three: ptr<ptr<ptr<int>>> = 42;
print(**two);
print(***three);
```

Each managed level owns a `ZL_ptr` value, not a raw C pointer. Therefore
`ptr<ptr<int>>` is not ABI-equivalent to C `int **`. Compatibility layers use
the native address field when adapting raw pointer-to-pointer APIs.

`ptr<ptr>` is accepted in an owned declaration when its missing inner type can
be inferred from the initializer. Public APIs should use the full type.

## Dereference and assignment

Every unary `*` removes exactly one `ptr` layer. Dereferencing `None`, a freed
owner, or a value with an incompatible runtime type produces a runtime error
instead of entering an invalid C address.

```zy
let *p: ptr<ptr<int>> = 10;
set **p = 20;
print(**p);
```

`ptr<void>` cannot be dereferenced. It must first be explicitly cast to a
concrete pointer type.

## Managed address identity

For a managed pointer, `&*p` yields an alias of `p`. It preserves the address,
ARC owner, ownership state, and runtime type tag. The rule composes:

```zy
let *source: ptr<ptr<int>> = 10;
let *slot: ptr<ptr> = &**source;
set **slot = 21;
print(**source); // 21
```

The alias retains its owner, so it remains valid even if the outer chain leaves
scope. This is a managed identity operation, not exposure of an untracked raw
interior address.

## Casts and `ptr<void>`

```zy
let opaque: ptr<void> = typed;
let restored: ptr<int> = (ptr<int>)opaque;
```

- `ptr<T>` implicitly converts to `ptr<void>`.
- `ptr<void>` to `ptr<T>` requires an explicit cast.
- Different concrete pointer types require an explicit cast.
- A cast does not change address, owner, ownership state, or runtime type tag.
- Function values cannot be cast to or from `ptr<T>`.

## Return and escape semantics

Managed values returned from a function transfer an owned reference to the
caller. Before local cleanup, the compiler retains the return value; normal
scope cleanup then releases local references.

```zy
fn make_value() -> ptr<int> {
    let *value: ptr<int> = 110;
    return value;
}
```

Local variable names are irrelevant: every call has independent storage and
ARC references. The same rule recursively covers nested pointers and managed
fields in returned structs.

```zy
struct Box {
    let this.value: ptr<int>;
}

fn make_box() -> Box {
    let *value: ptr<int> = 110;
    return Box { value: value };
}
```

All managed values reachable from the returned value are retained. Managed
locals that are not reachable from the return value are released at scope exit.

## Function values and closures

Function values use ABI v2 `ZL_Function { call, env, owner, signature }`.
Named top-level functions have no owner. Closure environments use atomic ARC,
and their destructor recursively releases captured function values, owned
pointers, and managed struct fields.

Parameters are borrowed for the duration of a call. Returning a managed value
transfers one owned reference. These rules also apply to values crossing a
`c_module` compatibility boundary.

## Explicit free

`mem.free(p)` invalidates an owned payload immediately for every alias:

- `0`: success;
- `-1`: `None`;
- `-2`: borrowed pointer;
- `-3`: already freed.

The ARC control block remains until the final alias is released. Automatic ARC
cleanup is preferred unless early invalidation is required.

## Safety boundaries

- ARC does not collect reference cycles.
- Borrowed pointers still follow the lifetime of their source.
- Pointer arithmetic is rejected for managed pointers.
- Managed pointers do not currently carry allocation bounds for indexing.
- External C libraries must use the ABI v2 retain/release and pointer adoption
  helpers when they save values beyond a call.

## Reference implementation

The reference implementation is ZyenLang v0.1.53. Its regression coverage is
in `tests/pointer_escape_test.zy`, `tests/nested_ptr_init_test.zy`, pointer and
memory tests, closure tests, and c_module callback tests.

## Copyright

This document is placed in the public domain under CC0-1.0.
