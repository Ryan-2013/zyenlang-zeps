# ZEP-0018 - Pointer 後綴優先序與 owned struct field

[English](ZEP-0018-pointer-postfix-and-owned-fields.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0018 |
| **標題** | Pointer 後綴優先序與 owned struct field |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-07-19 |
| **依賴** | [ZEP-0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md)、[ZEP-0015](ZEP-0015-function-cell-pointers.zh-TW.md) |
| **參考實作** | ZyenLang compiler main（v0.1.83 之後） |

## 摘要

本 ZEP 為 pointer、field、index、cast、call expression 建立一套可預測的
優先序，也把 owned pointer allocation 延伸到 struct field default。

## 優先序

後綴 field access、index、call 先於前綴 dereference、address-of、cast 結合；
括號用來選擇不同的 receiver。

```zy
set *car.value = 10;              // *(car.value)
set *((*car_ptr).value) = 20;     // *((*car_ptr).value)
let result: int = (*func_ptr)(1, 2);
```

當 `car_ptr` 型別是 `ptr<Car>` 時，`car_ptr.value` 無效；必須先用
`(*car_ptr).value` 選出 struct。編譯器診斷會直接提示這個寫法。

每個 `*` 只消耗一層 `ptr`。函式工廠指標使用：

```zy
let *factory_ptr: ptr<fn()->fn(int,int)->int> = add2;
let result: int = (*factory_ptr)()(20, 22);
```

`(**factory_ptr)` 無效，因為第一個 dereference 已經產生函式值；只有宣告型別
真的包含兩層 pointer 時才可使用兩個 `*`。

函式值呼叫鏈的 callee 與每個參數依 ZEP-0010 所有權規則各求值一次。

## Owned struct pointer field

Owned pointer field 在 `this` 前加 `*`，而且必須有具體 `ptr<T>` 型別與
pointee initializer：

```zy
struct Car {
    let *this.value: ptr<int> = 0;
}
```

每個 `Car{}` 都建立一個獨立、由 ARC 管理且初值為零的 `int` cell；field
本身的型別仍是 `ptr<int>`。讀寫與 method access 使用一般後綴規則：

```zy
let car: Car = Car{};
set *car.value = 10;

let *car_ptr: ptr<Car> = Car{};
set *((*car_ptr).value) = 20;
```

若 target 含巢狀 pointer，編譯器會配置每一個缺少的 layer：

```zy
struct Chain {
    let *this.value: ptr<ptr<int>> = 10;
}

let chain: Chain = Chain{};
print((str)(**chain.value));
```

Struct copy 會 retain field owner；assignment、return 與正常 scope exit 使用
ZEP-0014 的遞迴 managed-struct release helper。

普通 `let this.field: ptr<T>;` 維持一般 pointer field；literal 省略時為 `None`。
