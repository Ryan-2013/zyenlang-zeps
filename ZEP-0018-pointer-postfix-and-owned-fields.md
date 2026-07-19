# ZEP-0018 - Pointer Postfix Precedence and Owned Struct Fields

**English** | [Traditional Chinese](ZEP-0018-pointer-postfix-and-owned-fields.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0018 |
| **Title** | Pointer Postfix Precedence and Owned Struct Fields |
| **Authors** | zuenchen, OpenAI Codex |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-07-19 |
| **Requires** | [ZEP-0014](ZEP-0014-managed-pointers-and-arc.md), [ZEP-0015](ZEP-0015-function-cell-pointers.md) |
| **Reference implementation** | ZyenLang compiler main after v0.1.83 |

## Abstract

This ZEP gives pointer, field, index, cast, and call expressions one
predictable precedence model. It also extends owned pointer allocation to
struct field defaults.

## Precedence

Postfix field access, indexing, and calls bind before prefix dereference,
address-of, and casts. Parentheses select a different receiver.

```zy
set *car.value = 10;              // *(car.value)
set *((*car_ptr).value) = 20;     // *((*car_ptr).value)
let result: int = (*func_ptr)(1, 2);
```

`car_ptr.value` is invalid when `car_ptr` has type `ptr<Car>`; the struct must
first be selected with `(*car_ptr).value`. A compiler diagnostic suggests this
form.

Each `*` consumes exactly one `ptr` layer. A function factory pointer uses:

```zy
let *factory_ptr: ptr<fn()->fn(int,int)->int> = add2;
let result: int = (*factory_ptr)()(20, 22);
```

`(**factory_ptr)` is invalid because the first dereference produces a function
value. It only makes sense when the declared type contains two pointer layers.

The callee and every argument of a function-value chain are evaluated exactly
once under ZEP-0010 ownership rules.

## Owned struct pointer fields

An owned pointer field includes `*` before `this` and requires a concrete
`ptr<T>` type plus a pointee initializer:

```zy
struct Car {
    let *this.value: ptr<int> = 0;
}
```

Each `Car{}` creates a distinct ARC-owned `int` cell initialized to zero. The
field itself still has type `ptr<int>`. Reading, writing, and method access use
the normal postfix rules:

```zy
let car: Car = Car{};
set *car.value = 10;

let *car_ptr: ptr<Car> = Car{};
set *((*car_ptr).value) = 20;
```

If the target contains nested pointer layers, the compiler allocates every
missing layer:

```zy
struct Chain {
    let *this.value: ptr<ptr<int>> = 10;
}

let chain: Chain = Chain{};
print((str)(**chain.value));
```

Struct copies retain the field owner. Assignment, return, and all normal scope
exits use the recursive managed-struct release helpers from ZEP-0014.

Plain `let this.field: ptr<T>;` remains a normal pointer field and is `None`
when omitted from a literal.
