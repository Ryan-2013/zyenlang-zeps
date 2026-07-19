# ZEP-0015 - Managed Function Cell Pointers

**English** | [Traditional Chinese](ZEP-0015-function-cell-pointers.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0015 |
| **Title** | Managed Function Cell Pointers |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-07-18 |
| **Requires** | [ZEP-0010](ZEP-0010-first-class-functions.md), [ZEP-0014](ZEP-0014-managed-pointers-and-arc.md) |
| **Reference implementation** | ZyenLang compiler main after v0.1.83 |

## Abstract

This ZEP defines `ptr<fn(P...)->R>` as a managed pointer to a memory cell that
contains a first-class `ZL_Function` value. It provides pointer identity,
`ptr<void>` erasure, checked dereference, ARC-owned function cells, and
unambiguous indirect calls without converting a C function pointer to `void*`.

## Syntax

```zy
fn add(a: int, b: int) -> int {
    return a + b;
}

let pointer: ptr<fn(int,int)->int> = &add;
print((str)((*pointer)(20, 22)));
```

Postfix operations bind before prefix operations. A function pointer call is
therefore always written `(*pointer)(args)`. The removed `*pointer(args)` form
means `*(pointer(args))`; because a `ptr<fn(...)>` is not directly callable,
the compiler rejects it and suggests the parenthesized spelling.

## Storage

`&top_level_fn` points at a compiler-emitted static `ZL_Function` cell and is
borrowed. An owned heap cell uses the existing owned-pointer declaration:

```zy
let *owned: ptr<fn(int,int)->int> = add;
```

The cell retains a stored closure environment and releases it from the cell
destructor. Copies of the pointer follow ZEP-0014 ARC rules.

## Type erasure and checking

```zy
let opaque: ptr<void> = pointer;
let restored: ptr<fn(int,int)->int> = (ptr<fn(int,int)->int>)opaque;
print((str)((*restored)(12, 10)));
print((str)((*(ptr<fn(int,int)->int>)opaque)(12, 10)));
```

Erasure preserves the address, owner, ownership state, and runtime type tag.
Restoration requires an explicit cast. Calls validate the pointer's runtime tag
and the contained `ZL_Function.signature`. `None`, `Freed`, wrong source type,
and mismatched signatures produce errors before dispatch.

Direct casts between different `ptr<fn(...)>` signatures are forbidden.

## Function factory levels

A named function expression includes its own call layer. Given:

```zy
fn add1() -> fn()->fn(int,int)->int {
    return add2;
}
```

`add1` has type `fn()->fn()->fn(int,int)->int`, while `add1()` has type
`fn()->fn(int,int)->int`. The compiler does not insert an implicit call.

A pointer consumes exactly one dereference layer:

```zy
let *factory_ptr: ptr<fn()->fn(int,int)->int> = add2;
print((str)((*factory_ptr)()(20, 22)));
```

`(**factory_ptr)` is invalid because `*factory_ptr` is already a function
value. Double dereference requires `ptr<ptr<fn()->fn(int,int)->int>>`.

## C ABI

`ptr<fn(...)>` points to `ZL_Function` data storage. It is not a raw C function
pointer and is never converted to `void*`. Compatibility layers adapt raw C
callback signatures and continue to exchange `ZL_Function` values at the
ZyenLang ABI boundary.
