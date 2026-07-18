# ZEP-0011 — 結構體內方法定義

[English](ZEP-0011-struct-body-methods.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0011 |
| **標題** | 結構體內方法定義 |
| **作者** | zuenchen、Claude Opus 4.7 |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-06-21 |
| **修訂歷史** | 2026-06-21 |

## 摘要

ZyenLang 的 struct 方法**寫在 struct body 裡面**,跟在欄位宣告之後。
**沒有**外部 `fn StructName.method(...)` 的寫法。每個方法 body 都有
一個隱含的 `this`,型別是「指向該 struct 的指標」。這份 ZEP 把
v0.1.49 已經實作但沒明寫的規則正式固化。

## 動機

`struct` 宣告應該自我描述:讀者只看 struct body 就要能看到**全部**的
欄位**和全部**的方法。把方法簽章拆到別處的 `fn Repeater.run(...)`
會讓型別碎片化,而且讓使用者可以加上一個 struct 定義本身沒打廣告
的方法。C 那種「資料在這裡、操作在遠方」的傳統是有名的人體工學
缺點;ZyenLang 把它們收回同一塊。

## 規範

### 語法

```text
struct <Name> {
    let this.<field>: <type>;
    let this.<field>: <type> = <default>;     // ZEP-0006
    ...
    fn <method>( <params> ) -> <ret> {
        ...
        return <expr>;
    }
    fn <method>( <params> ) -> <ret> {
        ...
    }
}
```

- 慣例上欄位寫在方法前面;parser 不強制順序,但混雜寫會降低可讀性。
- 方法 header 跟頂層函式同文法(`fn name(<params>) -> <ret>`),
  含 [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md) 的 fn 型別
  參數與回傳型別。
- 方法可以完全沒有;只有欄位宣告的 struct 仍合法。
- 在 struct body **外面**寫 `fn StructName.method(...)` 是語法錯誤。
  沒有外部寫法。

### 隱含 `this`

在方法 body 裡,識別字 `this` 是「指向接收者 struct 實例」的指標,
**不**列在方法 header。C 端對應的函式名為 `<Struct>_<method>`,
第一個參數是 `<Struct>* this`。

```zy
struct Counter {
    let this.value: int;
    fn bump(by: int) -> int {
        set this.value += by;
        return this.value;
    }
}
```

編譯成:

```c
int Counter_bump(Counter* this, int by) {
    this->value += by;
    return this->value;
}
```

### 呼叫方法

不管接收者是值還是指標,語法一樣:

```zy
let c: Counter = Counter { value: 0 };
c.bump(5);
```

transpiler emit `Counter_bump(&c, 5)`。如果接收者是 `ptrstruct<Counter>`
就直接 emit `Counter_bump(c, 5)`。

方法可以透過 `this` 呼叫同 struct 的另一個方法:

```zy
struct Stats {
    let this.pass: int;
    let this.fail: int;
    fn check(cond: bool) -> int {
        if (cond) { set this.pass += 1; } else { set this.fail += 1; }
        return 0;
    }
    fn total() -> int {
        return this.pass + this.fail;
    }
}
```

### 禁止的構造

- **外部 `fn Struct.method(...)`** —— parse error,把方法搬進
  struct body。
- **省略 `fn` 的方法定義** —— 每個方法都以 `fn` 開頭。
- **重新指派 `this`** —— `set this = ...` 被拒;`this` 是唯讀指標。

## 設計理由

### 為什麼是 inline,不是外部

struct 定義就是使用者學「這個型別會做什麼」的標準位置。如果方法可以
寫在檔案任何地方,struct 定義就成了不完整的契約。外部寫法也誘惑
使用者複製 C++ 那種「header / implementation」分離,而 ZyenLang
明確不想要這個。

### 為什麼不要 `impl` 區塊

Rust 用 `impl StructName { fn ... }` 在 struct 定義後再加上方法。
那給了彈性(可以在別的模組加方法、可以針對 trait 寫一組),代價就是
ZyenLang 拒絕的那種間接層。v0.1.49 沒有 trait、沒有跨模組加方法,
所以 `impl` 區塊只會把 struct body 裡已經有的東西重述一次,加它只
會把文法弄大。

### 為什麼 `this` 不寫成參數

每個方法 header 都列 `this` 是 C 的樣板。ZyenLang 已經知道接收者
型別(就是外層 struct),指標性也是固定慣例。把 `this` 從 header
拿掉之後,方法簽章讀起來就跟使用者腦中對「這個操作」的模型一致。

## 向後相容性

完全相容 —— 本 ZEP 把 v0.1.49 已經實作的東西寫成規格。既有 inline
寫法的程式不變;若有人**嘗試**外部寫法,在 v0.1.49 就會吃到
`top-level statement is not allowed` 錯誤,不可能有現存程式靠它跑。

## 參考實作

實作在 `zyenlang-v0.1.49`(`zyenlang/transpiler.py`):

- `collect_signatures` 的 struct body 分支會把每個 struct body 內
  的 `fn <name>(...)` 註冊成頂層 fn,名稱為 `<Struct>_<name>`,
  第一個參數是 `this: ptrstruct<Struct>`。
- emit pass 鏡像同一個迴圈,emit
  `c_function_signature(<Struct>_<name>, ret_type, c_params) + " {"`
  再把 body 交給 `emit_function_body`。
- `transform_method_calls` 把 `obj.method(args)` 改寫成
  `Struct_method(&obj, args)`;如果 `obj` 本身就是
  `ptrstruct<Struct>` 則 emit `Struct_method(obj, args)`。

沒有獨立的 parser pass —— 方法跟 struct 同一輪解析。

## 待解問題

- 是否要加「靜態(associated)方法」—— 沒有 `this` 的方法。今天每個
  struct body fn 都有 `this`;若加 static 就跳過它。等實際需求出現
  再說。
- 是否要允許 struct body 內寫方法 prototype(只有 header 沒 body),
  之後在別處實作。目前拒絕;之後若加跨檔加方法再評估。

## 參考資料

- [ZEP-0006](ZEP-0006-struct-field-defaults.zh-TW.md) —— struct 欄位
  預設值,使用同一個 struct-body parse pass。
- [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md) —— fn 型別欄位,
  struct body 方法常透過 `this.field(args)` 呼叫它。

## 版權

本文件置於公有領域。
