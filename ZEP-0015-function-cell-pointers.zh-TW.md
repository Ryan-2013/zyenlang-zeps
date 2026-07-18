# ZEP-0015 - 受管函式 cell 指標

[English](ZEP-0015-function-cell-pointers.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0015 |
| **標題** | 受管函式 cell 指標 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-07-18 |
| **依賴** | [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md)、[ZEP-0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md) |
| **參考實作** | [ZyenLang v0.1.63](https://github.com/Ryan-2013/zyenlang/releases/tag/v0.1.63) |

## 摘要

本 ZEP 將 `ptr<fn(P...)->R>` 定義為指向 `ZL_Function` 記憶體 cell 的
managed pointer。它提供 pointer identity、`ptr<void>` 型別擦除、受檢查的
解參考、ARC owned function cell 與簡潔的間接呼叫，而且不會把 C function
pointer 強制轉成 `void*`。

## 語法

```zy
fn add(a: int, b: int) -> int {
    return a + b;
}

let pointer: ptr<fn(int,int)->int> = &add;
print((*pointer)(20, 22));
print(*pointer(20, 22));
```

`(*pointer)(args)` 是加括號形式；`*pointer(args)` 是等價的 ZyenLang 簡寫。
因此零參數函式指標可以寫成 `*pointer()`。

## 儲存與所有權

`&top_level_fn` 指向編譯器產生的靜態 `ZL_Function` cell，屬於 borrowed
pointer。owned heap cell 使用既有的 owned-pointer 宣告：

```zy
let *owned: ptr<fn(int,int)->int> = add;
```

cell 會 retain 存入的 closure environment，並在 cell destructor 中 release。
pointer 複製遵守 ZEP-0014 的 ARC 規則。

## 型別擦除與檢查

```zy
let opaque: ptr<void> = pointer;
let restored: ptr<fn(int,int)->int> = (ptr<fn(int,int)->int>)opaque;
print(*(ptr<fn(int,int)->int>)opaque(12, 10));
```

擦除不改變 address、owner、ownership state 或 runtime type tag。還原需要顯式
cast。呼叫會同時檢查 pointer runtime tag 與 cell 內的
`ZL_Function.signature`。`None`、`Freed`、錯誤原始型別及簽章不符都會在
dispatch 前報錯。

不同 `ptr<fn(...)>` 簽章禁止直接互相 cast。

## 函式工廠的層級

具名函式 expression 包含函式本身的一層呼叫。以下例子中：

```zy
fn add1() -> fn()->fn(int,int)->int {
    return add2;
}
```

`add1` 的型別是 `fn()->fn()->fn(int,int)->int`；`add1()` 才是
`fn()->fn(int,int)->int`。編譯器不會偷偷插入一次函式呼叫。

## C ABI

`ptr<fn(...)>` 指向 `ZL_Function` data cell，不是 raw C function pointer，
也不會被轉成 `void*`。C 相容層負責適配原生 callback signature，並繼續在
ZyenLang ABI 邊界交換 `ZL_Function` 值。
