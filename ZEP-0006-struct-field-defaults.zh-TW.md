# ZEP-0006 — 結構欄位預設值

[English](ZEP-0006-struct-field-defaults.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0006 |
| **Title** | Struct Field Defaults |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

struct 欄位宣告可以帶一個尾端預設運算式。每次建構這個 struct 時都會
評估這個運算式;凡是 struct literal 沒有明確給值的欄位(或宣告無初值的
情況下所有欄位)都套用宣告的預設值。沒有預設值的欄位維持原本 C99 零填
的行為。

## 動機

在這份 ZEP 之前,要給 struct 欄位非零預設值的唯一辦法是寫一個建構函式:

```zy
struct Car {
    let this.model: str;
    let this.year: int;
}

fn new_car(model: str) -> Car {
    return Car { model: model, year: 2024 };
}
```

可行,但強迫每個呼叫者都走 helper,而且預設值的文件位置跟欄位宣告分開。
這個 pattern 太常見,而且 v0.1.49 對函式參數已經支援平行機制
(`fn f(a: int, b: int = 1) -> int`)。本 ZEP 把同樣的概念延伸到 struct 欄位,
沒有新語法形式 —— 只是在欄位型別後面加可選的 `= 表達式`。

## 規範

### 語法

struct 欄位文法從

```text
let this.<name> : <type> ;
```

擴充為

```text
let this.<name> : <type> ( = <expression> )? ;
```

沒帶預設值的欄位宣告不變。

預設 `<expression>` 在周圍源碼語境下解析,可以使用任何函式 body 接受的
運算式,**唯一例外**是不能參照同 struct 的其他欄位(`this.other` 在欄位
宣告時不在 scope 內)。模組層級常數函式、數值/字串/布林字面值、算術運算
都行。

### 建構點套用順序

編譯器在三種建構情境套用預設:

1. **帶欄位列表的 struct literal** —— `Car { model: "Tesla" }`:
   literal 裡的欄位贏;省略但宣告有預設的欄位用預設值填;其餘欄位由 C99
   補零。

2. **空 struct literal** —— `Car {}`:所有有預設的欄位填上預設值;其餘
   補零。

3. **宣告無初值** —— `let c: Car;` 與建構式形式 `let c: Car = Car;`:
   跟空 literal 一樣。

在情境 (1) 中,明確欄位與預設填入的欄位在產生的 C 裡可以任意順序,
因為 C99 designated initializer 是順序無關的。

### 範例

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

### 與欄位宣告順序的互動

當欄位都有預設值且都沒被 literal override,套用順序遵循 struct 宣告的
順序。生成的 C 用 designated initializer,所以順序只是為了可讀性,不影響語意。

### 禁止 / 範圍外

- 預設運算式**不可**參照 `this.<其他欄位>`。跨欄位依賴會需要排序欄位初始
  化,本 ZEP 刻意避開這個複雜度。
- 預設運算式**不可**呼叫一個自己又建構同型 struct 的函式(不能遞迴)。
- 預設運算式**不可**依賴 side effect(例如全域可變狀態 —— ZyenLang 依
  [ZEP-0004](ZEP-0004-global-state-and-constants.zh-TW.md) 沒有全域可變
  狀態,所以這條自動成立)。

## 設計理由

### 為什麼沒有順序約束

函式參數預設必須在尾端,因為位置呼叫需要明確的填補順序。Struct literal
按欄位名 keyed,沒有位置形式,所以任何欄位都可以帶預設值而不影響其他。
意思是既有 struct 宣告可以漸進加預設值,不用重排。

### 為什麼是運算式,不只是字面值

預設值會走 `transform_expr`,所以可以用模組層級常數函式、簡單算術、字串
串接 —— 任何 fn body 內 `let` 初始化能接受的東西。這是免費的,因為
降階路徑早就存在,讓你可以重用宣告過的常數而不是重複數值。

### 為什麼禁掉 `this.other_field`

維持降階的簡單性。跨欄位預設會強迫編譯器拓樸排序初始化器並偵測循環,
為了一個用 method(`fn area() -> float`)就能等價解決的功能
(`let this.area: float = this.width * this.height;`)增加實質複雜度,
不值得。

### 為什麼第一版就 Final

特性小、語意被 C99 designated initializer 行為釘住、實作已 ship。
立刻鎖到 `Final` 是表明語法不接受 bikeshedding;未來變更走新 ZEP 來
Supersede 這份。

## 向後相容性

完全向後相容。

- 沒有 `= default` 的 struct 欄位宣告解析方式跟以前一樣。
- 提供所有欄位的 struct literal 產生跟以前相同的 C。
- 省略欄位**而且** struct 沒宣告預設值的 literal 仍然產生 C99 補零行為。
- 唯一新生的程式碼是有預設值欄位的 `.field = expr` 條目,本來就是合法 C99。

`tests/` 裡沒有任何既有測試需要修改;所有 14 個 `text_test` cases 仍然
通過,`examples/add.zy` 不受影響。

## 參考實作

- `zyenlang/transpiler.py`:
  - `StructDef` 多一個 `defaults: Dict[str, str]` 欄位。
  - `collect_signatures()` 的欄位 regex 接受尾端可選 `= <expression>`。
  - emit pass 中對映的 regex 也接受相同形式。
  - `convert_struct_literal()` 用 struct 宣告的預設值填補沒明確指定的欄位。
  - 新 helper `_struct_default_init(struct_name, ctx)` 給 `parse_var_decl()`
    的「宣告無初值」(`let c: Car;`)與「建構式形式」(`let c: Car = Car;`)
    兩種情況使用。

## 待釐清問題

- 是否該風格上推薦 `Car {}` 字面值(大括號內無內容)而不是 `let c: Car;`。
  兩者產生相同 C;本 ZEP 視為可互換。
- 是否該把同樣語法延伸到函式參數的 `*args` / `**kwargs`(不能 —— 見 ZEP /
  memory note 拒絕 variadic kwargs)。

## 參考

- [ZEP-0004](ZEP-0004-global-state-and-constants.zh-TW.md) —— struct
  singleton 存在的結構性理由;本 ZEP 讓它在使用上順手。
- v0.1.49 函式預設參數(主 repo README)—— 本 ZEP 鏡像的前例。
- C99 designated initializers, §6.7.8 —— 讓未指定欄位零填免費的底層 C 特性。

## 版權

本文件置於公有領域。
