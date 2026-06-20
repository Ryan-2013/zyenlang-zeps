# ZEP-0003 — ZyenLang 風格指南

[English](ZEP-0003-style-guide.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0003 |
| **Title** | ZyenLang Style Guide |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Informational |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

`.zy` 原始碼的日常風格規則。等同 ZyenLang 的 PEP 8,但短一點,因為語法表面
也小一點。

本 ZEP 是 `Informational`:編譯器不強制。標準函式庫遵守這些規則;使用者程式碼
也應該遵守,但不遵守也不會被拒絕。

## 縮排與空白

- **4 個空格**一層縮排。原始碼裡不要 tab。
- 檔案層級每兩個函式之間空一行。
- struct 內每兩個 method 之間空一行。
- 行尾不留空白。
- 檔案結尾留一個換行。

## 行的排版

- 一行一個邏輯語句。ZyenLang parser 允許在 `(...)`、`[...]`、`{...}` 內換行,
  但**不允許**同一行兩個語句。
- 行寬上限 **100 欄**。長的函式呼叫與簽章用多行形式拆開:

  ```zy
  fn long_name(
      first_arg: int,
      second_arg: str,
      third_arg: List
  ) -> int {
      return process(
          first_arg,
          second_arg,
          third_arg
      );
  }
  ```

- `if` / `else` / `for` 一律帶大括號,即使 body 只有一行。ZyenLang parser
  不能穩定接受單行 `if (x) { stmt; }`,**永遠拆成多行**:

  ```zy
  // YES
  if (n > 0) {
      set n -= 1;
  }

  // NO —— v0.1.49 parser 不穩
  if (n > 0) { set n -= 1; }
  ```

## 命名

| 種類 | 慣例 | 範例 |
|---|---|---|
| 函式 | `snake_case` | `load_buffer`、`classify_word` |
| 區域變數 | `snake_case` | `cursor_line`、`hi_lines` |
| 函式參數 | `snake_case` | `bit_index`、`cursor_col` |
| Struct 型別 | `PascalCase` | `Counter`、`Stack`、`StringMap` |
| Struct 欄位 | `snake_case` | `this.base`、`this.list_len` |
| 模組名 | `lowercase`,理想上單字 | `string`、`path`、`fs`、`tk` |
| 模組層級常數 fn | `snake_case`,讀起來像名詞 | `max_pwm()`、`default_port()` |
| 列舉式常數 fn | `SCREAMING_SNAKE` 可 | `KIND_KEYWORD()`、`KIND_PLAIN()` |
| 檔名 | `snake_case.zy` | `tk_session_test.zy` |

模組層級的「常數」需要特別說明:ZyenLang 不允許頂層 `let` / `const`
(見 [ZEP-0004](ZEP-0004-global-state-and-constants.zh-TW.md)),所以常數
寫成零參數函式。命名上**取它回傳的名詞**,不要當 getter(`max_pwm`,
不是 `get_max_pwm`)。

當常數本質上是一組列舉值時,用 `SCREAMING_SNAKE`(`KIND_KEYWORD()`、
`MODE_READ()`)。一般數值常數用 `snake_case`。

## 型別標註

- **一律**標註函式參數與回傳型別。文法上必填。
- **一律**標註 struct 欄位。文法上必填。
- **建議**在 `let` 宣告也標,即使型別顯而易見:`let n: int = 0;` 在一段
  又有 float 又有 ptr 的 body 裡比 `let n = 0;` 好讀。
- 例外:右邊是建構式呼叫、名字已經明示型別時,標註就是雜訊:

  ```zy
  // 要標
  let cursor_line: int = 0;
  let visible: bool = true;
  let path: str = "main.zy";

  // 可省 —— 右邊自己講清楚了
  let xs: List;
  let m: StringMap = map.new();
  ```

## Import

- 一行一個 import。
- 排序:標準函式庫先,使用者相對 import 後。
- 兩組之間隔一個空行。
- **一律**寫 `as alias`,即使 alias 跟模組名相同。這讓呼叫端統一,
  也避免 import 之後改名造成命名衝突:

  ```zy
  import <std/string>;          // string.len(s)         OK
  import <std/string> as string; // 一樣意思,但明確 —— 推薦
  import <std/text> as text;
  import <std/fs> as fs;

  import "lex.zy" as lex;
  import "complete.zy" as complete;
  ```

  IDE 的 `Ctrl+I` 自動補 import 工具(`ide_gui.zy`)會自動套用這條慣例 ——
  不確定就跑一下。

## 修改

- 每次修改都要 `set` 關鍵字。編譯器強制這條;風格上,**把 `set` 放在行首**,
  即使在 `for` header 第三段也一樣:

  ```zy
  // 推薦
  for (let i: int = 0; i < n; set i += 1) { ... }

  // 也接受 —— for-step 裡 `i += 1` 不寫 set 也能編
  for (let i: int = 0; i < n; i += 1) { ... }
  ```

  兩種都能編。明寫 `set` 在視覺上更清楚是個修改動作。

## 函式形狀

- 一個函式做一件事。body 超過 ~40 行就拆 helper。
- 鼓勵 early-return guard:

  ```zy
  fn process(buf: List) -> int {
      if (buf.len() == 0) {
          return 0;
      }
      if (!validate(buf)) {
          return -1;
      }
      // …主流程…
  }
  ```

- 避免深層巢狀 `if`。三層是 code smell,四層是還沒爆的 bug。

## Struct

- 一個 struct 對應一個概念。不要硬把東西塞一起。
- 欄位宣告在前,method 在後:

  ```zy
  struct Counter {
      let this.base: int;
      let this.step: int;

      fn next() -> int {
          set this.base = this.base + this.step;
          return this.base;
      }
  }
  ```

- 只放資料、沒有 method 的 struct 完全 OK 而且很正常。

## 註解

- 預設**不寫**註解。命名好、body 清楚的函式不需要註解。
- 當「為什麼」不明顯時才寫:繞過某個 compiler quirk、某個非平凡 invariant、
  某個效能 trade-off。
- 不要重述程式碼已經顯示的「做什麼」:

  ```zy
  // YES —— 抓住了非顯而易見的 why
  // String concat in a loop is O(N^2); use string.replace which is O(N).
  let normalized: str = string.replace(s, "\r\n", "\n");

  // NO —— 重述程式碼本來就顯示的事
  // Set n to 0
  let n: int = 0;
  ```

## 編譯器強制 vs. 本 ZEP 推薦

| 規則 | 來源 |
|---|---|
| 修改要寫 `set` | **編譯器** |
| `fn` 與 `struct` 欄位的型別標註 | **編譯器** |
| 不允許頂層 `let` / `const` | **編譯器**,見 [ZEP-0004](ZEP-0004-global-state-and-constants.zh-TW.md) |
| 本 ZEP 列的命名慣例 | **風格指南**(不強制) |
| 4 空格縮排 | **風格指南** |
| 100 欄行寬 | **風格指南** |
| 每個 import 加 `as alias` | **風格指南** |

## 待釐清問題

- 是否該有 `zy fmt` 工具?大概有一天會,但要等表面更被打磨過再說。
- 模組層級數值常數該用 `snake_case` 還是 `SCREAMING_SNAKE`,現行做法兩者
  混用,上面那張表是指引,不是硬規矩。

## 版權

本文件置於公有領域。
