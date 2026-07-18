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
| **Reference implementation** | [ZyenLang v0.1.63](https://github.com/Ryan-2013/zyenlang/releases/tag/v0.1.63) |

## Abstract

This ZEP defines `ptr<fn(P...)->R>` as a managed pointer to a memory cell that
contains a first-class `ZL_Function` value. It provides pointer identity,
`ptr<void>` erasure, checked dereference, ARC-owned function cells, and concise
indirect calls without converting a C function pointer to `void*`.

## Syntax

```zy
fn add(a: int, b: int) -> int {
    return a + b;
}

let pointer: ptr<fn(int,int)->int> = &add;
print((*pointer)(20, 22));
print(*pointer(20, 22));
```

`(*pointer)(args)` is the parenthesized form. `*pointer(args)` is equivalent
ZyenLang shorthand. A zero-argument pointer call may therefore be written
`*pointer()`.

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
print(*(ptr<fn(int,int)->int>)opaque(12, 10));
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

## C ABI

`ptr<fn(...)>` points to `ZL_Function` data storage. It is not a raw C function
pointer and is never converted to `void*`. Compatibility layers adapt raw C
callback signatures and continue to exchange `ZL_Function` values at the
ZyenLang ABI boundary.
