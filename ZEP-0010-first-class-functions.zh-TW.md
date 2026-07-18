# ZEP-0010 — 一等公民函式值

[English](ZEP-0010-first-class-functions.md) | **繁體中文**

> **注記(2026-06-21):**「禁止巢狀函式定義」這條規則已被
> **[ZEP-0013](ZEP-0013-closures.zh-TW.md) 取代** —— 巢狀 `fn`
> 現在合法,會被 lambda-lift 成閉包,環境配置在 heap、捕獲是唯讀。
> 本 ZEP 其餘規則仍然有效。自 ABI v2 起，fn 值使用
> `ZL_Function { call, env, owner, signature }`；裸具名 fn 沒有 environment
> owner，closure 則 retain 一個 ARC-managed environment。詳見
> [ZEP-0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md)。

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0010 |
| **標題** | 一等公民函式值 |
| **作者** | zuenchen、Claude Opus 4.7 |
| **狀態** | Active |
| **類型** | Standards |
| **建立日期** | 2026-06-21 |
| **修訂歷史** | 2026-06-21 |

## 摘要

ZyenLang 新增 `fn(<型別清單>)-><型別>` 型別表示式,讓「已具名的頂層
函式」可以被當作值傳遞。`fn` 型別的值可以放進變數、當參數傳入、從
函式回傳,也可以做結構欄位。**它不能在執行期被合成**:沒有 lambda、
沒有閉包、沒有捕獲外層變數。`fn` 型別值的唯一合法來源就是參考一個
已經定義好的頂層函式(或從這類值衍生而來的另一個 `fn` 型別值)。

## 動機

使用者一再遇到「要傳的是動作本身,不是動作的結果」的需求 —— 比較器、
歸納器、分派表、策略選擇。在本 ZEP 之前,只能靠以下兩種方式繞:

1. 在被呼叫的函式裡寫死 `if/else` 字串或列舉識別碼。
2. 每個情況各寫一個特化函式。

兩者都會把程式碼撐大,而且擴充很麻煩。主流語言對這個需求的解法是
函式指標(C)、函式參考(C++ `std::function`)、或完整閉包
(JavaScript、Python)。使用者明確不要 closure 跟 lambda 語法
(見 [ZEP-0003](ZEP-0003-style-guide.zh-TW.md) 以及記憶筆記
`zyenlang-syntax-no-complex`)。C 風格函式指標是中庸之道:傳遞
**具名**函式的位址,僅此而已。

## 規範

### 型別語法

```text
fn ( <參數型別> ( , <參數型別> )* ) -> <回傳型別>
fn ( ) -> <回傳型別>
fn ( <參數型別> ( , <參數型別> )* )            # 隱含 -> void
fn ( )                                          # 隱含 -> void
```

- 參數型別只列出型別,**禁止寫參數名稱**:寫
  `fn(int,int)->int`,不要寫 `fn(a:int,b:int)->int`。名稱屬於
  「函式定義」,不屬於「函式型別」。
- 回傳型別可以是 ZyenLang 任何一般型別,包括另一個 `fn(...)->T`。
  v0.1.49 支援巢狀 1 層;在「參數位置」更深層的巢狀屬於本 ZEP
  範圍外。
- 一般型別能容忍的空白,這裡也可以放。

### 型別可以出現的位置

`fn(...)->T` 型別表示式可以出現在任何「內建型別」可以出現的地方:

```zy
// 當參數型別
fn apply(op: fn(int,int)->int, x: int, y: int) -> int { ... }

// 當回傳型別
fn pick(name: str) -> fn(int,int)->int { ... }

// 當區域變數型別
let f: fn(int,int)->int = add;
let g: fn(int)->int;             // 未指派前預設 NULL

// 當結構欄位型別
struct Reducer {
    let this.op: fn(int,int)->int;
    let this.seed: int;
}
```

### 值從哪裡來

對 `fn` 型別的指派、參數、欄位來說,合法的右邊只可能是:

1. 對具名頂層函式的裸引用,如 `add`、`sub`。
   (`MyStruct.method` 留給未來,v0.1.49 不支援。)
2. 回傳型別是 `fn(...)->T` 的函式呼叫表達式。
3. `NULL` 不可在源碼裡明寫;沒給初始值的宣告會自動以 NULL 起始。

`fn` 型別變數若仍是 NULL,呼叫前必須先指派。對 NULL `fn` 值呼叫
是未定義行為(C backend 會直接 crash)。

### 呼叫 `fn` 型別值

語法和直接呼叫一樣:

```zy
let f: fn(int,int)->int = pick("add");
let r: int = f(3, 4);             // 生成 C 的 `f(3, 4)`
```

透過結構欄位:

```zy
fn Reducer.run(values: List) -> int {
    set acc = this.op(acc, v);    // 生成 C 的 `this->op(acc, v)`
}
```

### 型別檢查

兩個 `fn` 型別相等的條件是:正規化空白後,參數清單與回傳型別都相等。
`fn(int,int)->int` 不能指派給 `fn(int,int)->float`。

呼叫端的參數不會在原本的「直接呼叫」規則之外額外轉換 —— C 編譯器
會強制函式指標的簽章,簽章不合則由 backend 報錯。

### 禁止的結構

- **巢狀函式定義** —— `fn outer() { fn inner() {...} }` 是編譯
  期錯誤。編譯器會報:

  ```text
  line N: nested function definitions are not allowed; define `fn` at
  file scope and use a struct field of fn type to bind state, e.g.
  `let this.f: fn(int)->int;`
  ```

  這條規則是承重的。它保證執行期永遠不必存在任何「捕獲的環境」,
  所以整個 runtime 等價於一張 C 函式指標表。

- **`fn` 型別裡寫參數名** —— 見上面型別語法段。

- **匿名函式字面值** —— v0.1.49 沒有 `fn(x){...}` 表達式語法,
  本 ZEP 不打算加。

## 設計理由

### 為什麼用函式指標,不用閉包

閉包會捕獲外層作用域。要把閉包編譯到 C,編譯器得:(a)決定要捕獲
哪些外層變數,(b)在堆配置環境紀錄,(c)把它跟函式指標包成胖值,
(d)當閉包不可達時釋放環境。每一步都會逼 ZyenLang 走向 GC 或借用
檢查器,兩者都不在 v0.1.49 範圍內,而且會直接打掉「C-like 表面」
這個目標。

具名頂層函式沒有這些問題。它們本來就被編譯成普通 C 函式;它們的
位址在程式整個生命週期都有效;什麼都不用配置。

### 為什麼禁止型別裡寫參數名

`fn(a:int,b:int)->int` 和 `fn(int,int)->int` 是同一個型別 —— 名字
只跟函式的*定義*有關。允許在型別表達式裡寫名字,等於邀請使用者
為同一組參數寫三套名稱(定義、宣告、型別),它們會慢慢漂走。在
parse 階段禁掉,可以保證只有一個 source of truth。

### 為什麼禁止巢狀 `fn`

只要允許 `fn` 內套 `fn`,下一個顯然的需求就是讓內層引用外層的
區域變數。那就是閉包。禁止它就讓規則機械化:每個 `fn` 都住在
檔案層級,跟今天頂層函式的規則一樣,不引入新的 scope 概念。

### 為什麼用 struct 欄位當閉包替代

經典的「我要把狀態跟函式綁在一起」變成:

```zy
struct Reducer {
    let this.op: fn(int,int)->int;
    let this.seed: int;
}
```

使用者*把環境寫成具名欄位*,而不是讓編譯器從變數使用情況推斷。
這就是 Go「方法掛在 struct 上」 vs. JavaScript「自由閉包」的取捨。
struct 版本比較囉嗦,但是自我解釋,而且零隱性配置。

## 向後相容性

完全相容。

- 在本 ZEP 之前,`fn(` 的 token 序列只有出現在頂層宣告或結構方法
  的開頭才有意義。現有程式碼不會在 `:` 或 `->` 之後接 `fn(`,所以
  沒有現有程式因此變模糊。
- 既有的結構欄位預設值([ZEP-0006](ZEP-0006-struct-field-defaults.zh-TW.md))
  依然適用;沒有預設值的 `fn` 型別欄位以 NULL 起始(C99 zero-fill),
  是安全的哨兵值。

## 參考實作

目前參考實作在 ZyenLang v0.1.53（`zyenlang/transpiler.py`）：

- **輔助函式** —— `is_fn_type`、`parse_fn_type`、`fn_typedef_name`、
  `register_fn_typedef`、`emit_fn_typedefs`、`collect_fn_typedefs`、
  `parse_fn_header_line`。
- **型別文法** —— 新增模組層常數 `TYPE_RX`,同時接受簡單型別與
  fn 型別;在結構欄位、`let` 宣告、`parse_var_decl`、結構成員
  emit pass 共用。
- **`c_type`** 把 `fn(...)->T` 映射為決定性的 typedef 名稱
  `ZL_fn_<params>_to_<ret>`。
- **`validate_user_type`** 對 fn 型別的各片段遞迴驗證,並擋掉
  「型別裡寫參數名」。
- **`infer_type`** 對裸的具名函式參考回傳對應的 fn 型別;對「透過
  fn 型別變數呼叫」也能解析。
- **`transform_method_calls`** 偵測 fn 型別欄位,emit
  `obj.field(args)` / `obj->field(args)`,不再走 `Struct_method(&obj, ...)`
  分派路徑。
- **`emit_function_body`** 對任何巢狀 `fn ` 立刻報錯。
- **代碼產出順序** —— 先 emit 結構的 forward declarations、再
  emit fn typedefs(可以參考結構名)、再 emit 結構完整定義(裡面
  可能含 fn 型別欄位,使用 typedefs)、再 emit 函式 prototypes。

測試在 `tests/fn_value_test.zy`,覆蓋全部四種使用情境(參數、回傳、
結構欄位、未初始化先宣告再指派)。

## 待解問題

- 是否要為「無參數、回傳 void」這種 callback 加個簡寫。目前必須寫
  `fn()->void`;有些語言允許只寫 `fn()`。本 ZEP 已經接受省略
  `->void`,但官方風格指南會挑出唯一推薦寫法。
- 是否要允許 `fn` 型別在任意巢狀位置出現
  (`fn(fn(int)->int)->fn(int)->int`)。實作工作不大(正規式
  / parser 加深),問題是讀起來夠不夠清楚。
- 方法參考 —— `let m = some_struct.method;` 目前不支援;struct
  方法一定要透過隱含的 `this`。未來的 ZEP 可能會加上「綁定方法
  值」。

## 參考資料

- [ZEP-0003](ZEP-0003-style-guide.zh-TW.md) —— 表面語法的克制原則。
- [ZEP-0004](ZEP-0004-global-state-and-constants.zh-TW.md) —— struct
  singleton 作為現有的「綁定狀態而不用全域變數」模式;fn 型別欄位
  把這個模式普及化。
- C99 §6.7.5.3(Function declarators)與 §6.5.2.2(Function calls)
  —— 本 ZEP 編譯到的 C 語意。

## 版權

本文件置於公有領域。
