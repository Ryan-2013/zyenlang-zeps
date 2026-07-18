# ZEP-0016 - Structural List Method Dispatch

**English** | [Traditional Chinese](ZEP-0016-structural-list-dispatch.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0016 |
| **Title** | Structural List Method Dispatch |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-07-18 |
| **Requires** | [ZEP-0011](ZEP-0011-struct-body-methods.md), [ZEP-0014](ZEP-0014-managed-pointers-and-arc.md) |
| **Reference implementation** | [ZyenLang v0.1.83](https://github.com/Ryan-2013/zyenlang/releases/tag/v0.1.83) |

## Abstract

This ZEP allows heterogeneous Lists to store user-defined struct values and
call a common method without declaring an interface, trait, inheritance tree,
public union, or public `Any` type. Compatibility is determined structurally
from the method name and exact function signature.

## Syntax

```zy
struct Dog {
    fn bark() -> void {
        print("dog");
    }
}

struct Cat {
    fn bark() -> void {
        print("cat");
    }
}

fn main() -> int {
    let animals: List = [Dog {}, Cat {}];
    for (let i = 0; i < animals.len(); set i += 1) {
        animals.get(i).bark();
    }
    return 0;
}
```

## Static compatibility

The compiler conservatively records the element types introduced into a named
local List by its literal and direct `append`, `append_ptr`, and `set`
operations. A structural call is accepted only when every possible element is
a user-defined struct and every struct provides the requested method with the
same parameter types, return type, and default expressions.

Structural calls use positional arguments. Parameter names are not part of
the shared shape. Missing methods, scalar elements, and signature differences
are compile-time errors while the possible type set is known.

## Erased List boundaries

A plain `List` parameter erases the closed element set because the callee may
mutate the List. A structural call on an erased List is compiled only when the
program has one unambiguous signature for that method name. Generated code
checks the boxed runtime type before dispatch. An incompatible value reports a
runtime error instead of calling an invalid C function.

## Storage and ownership

ABI v3 adds a boxed struct variant to internal `Any` values. Appending a struct
copies it into an ARC-owned box and recursively retains managed fields.

- `get()` returns a borrowed view of the List cell.
- Copying that view into another List retains the box.
- `set()` and `clear()` release removed boxes.
- `pop()` transfers the box reference.
- `(StructType)value` checks the exact runtime type before restoring a value.

Method receivers point at the boxed value, so a mutating method updates the
List element. The original struct used to initialize the List remains a
separate value copy.

## Boundaries

External c_module structs are not boxed directly. A user-defined ZyenLang
facade struct establishes ownership and methods at the language boundary.
List still rejects bare `fn(...)` values; store the function in a user struct
field when a List-held callback is required.

List itself is not ARC-managed and ARC does not collect cycles. Programs
should call `clear()` when deterministic release of managed elements matters.
