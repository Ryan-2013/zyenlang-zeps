# ZEP-0004 — 全域狀態與模組層級常數

[English](ZEP-0004-global-state-and-constants.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0004 |
| **Title** | Global State and Module-Level Constants |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Final |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

ZyenLang **拒絕頂層 `let` 與 `const` 宣告**。一個程式的檔案層級只能有
`import`、`struct`、`fn`。本 ZEP 解釋這條規則、為什麼存在、以及取代其他語言
全域變數用途的四個慣用寫法:模組層級常數、唯讀設定、可變共享狀態、跨呼叫
持久狀態。

## 動機

早期 ZyenLang 版本會**靜默丟掉**頂層 `let` 宣告:parser 接受,但從不 emit
到 C,結果參照它們的程式碼會在 gcc 階段失敗於 `undeclared identifier`。對使
用者來說是 P2 等級的 bug,對從 Python 或 JavaScript 移植過來的人是個漂亮
陷阱。

v0.1.49 把兩邊都堵起來:編譯器明確**拒絕**頂層 `let` / `const` 並給出清楚
錯誤訊息;本 ZEP 把替代的四個寫法固化下來。

## 規範

### 規則本體

檔案層級只能有以下構造:

```text
import <std/...>;
import "..." as ...;
struct Name { ... }
fn name(...) -> T { ... }
```

下列在檔案層級會在編譯期被拒絕:

```zy
let g: int = 1;       // 錯誤
const max_pwm = 1000; // 錯誤
set g = 2;            // 錯誤 —— `set` 本來就不能在 fn body 外
```

可變變數只能存在於 `fn` body 內。常數寫成零參數 `fn`。共享狀態用顯式傳遞,
或封裝進 struct,或寫到 disk。

### Pattern 1 —— 模組層級常數

數值或字串常數在整個程式裡都一樣時,寫成**零參數函式**暴露出來:

```zy
fn max_pwm() -> int {
    return 1000;
}

fn default_port() -> int {
    return 8080;
}

fn app_name() -> str {
    return "ZyenLang IDE";
}
```

呼叫端看起來就跟普通函式呼叫一樣:

```zy
let pwm: int = clamp(target, 0, max_pwm());
```

這是**首選**慣用寫法,第一個想到的就用這個。

#### 子模式:列舉式常數

一群常數構成列舉時,用共同前綴命名:

```zy
fn KIND_PLAIN()   -> int { return 0; }
fn KIND_KEYWORD() -> int { return 1; }
fn KIND_TYPE()    -> int { return 2; }
fn KIND_NUMBER()  -> int { return 3; }
```

`std/text/lex.zy`(IDE 的語法 highlighter)的 `KIND_*` codes 就是這樣定義的。
命名慣例見 [ZEP-0003](ZEP-0003-style-guide.zh-TW.md)。

### Pattern 2 —— 從 disk 讀唯讀設定

對於會因環境而異的值(路徑、port、feature flag),啟動時用 `std/config`
讀一次:

```zy
import <std/config>;

fn load_settings() -> StringMap {
    return config.load("settings.ini");
}

fn main() -> int {
    let cfg: StringMap = load_settings();
    let port: int = string.to_int(cfg.get("port", "8080"));
    serve(port);
    return 0;
}
```

呼叫端把 `cfg`(或拆出來的 `port`)顯式往下傳。不需要檔案層級狀態。

### Pattern 3 —— 用 struct singleton 表達可變共享狀態

當好幾個函式真的需要共享可變狀態時,宣告一個 struct,在 `main` 裡實例化一次,
然後**用 reference 傳**(或塞進 `List` cell)給每個用到的函式:

```zy
struct AppState {
    let this.cursor_line: int;
    let this.cursor_col: int;
    let this.dirty: bool;

    fn mark_dirty() -> void {
        set this.dirty = true;
    }
}

fn handle_keystroke(state: AppState, key: str) -> void {
    if (string.eq(key, "Right")) {
        set state.cursor_col += 1;
        state.mark_dirty();
    }
}

fn main() -> int {
    let state: AppState = AppState { cursor_line: 0, cursor_col: 0, dirty: false };
    handle_keystroke(state, "Right");
    return 0;
}
```

比用 global 囉嗦,但資料流看得到。ZyenLang IDE(`apps/zyide_gui.zy`,
~2500 行)整支就是這個 pattern。

### Pattern 4 —— 跨呼叫、跨執行的狀態用 disk 持久化

狀態需要跨函式呼叫**而且**跨程式執行存活時,寫到檔案。`std/log` 就是這樣做:

```zy
import <std/fs>;

fn log_path() -> str {
    return ".zy_log_file";
}

fn write_log(message: str) -> int {
    return fs.append(log_path(), message + "\n");
}

fn read_log() -> str {
    return fs.read(log_path());
}
```

這裡的「global」是 disk 上的檔案,不是 process 記憶體裡的變數。優點:撐得過
crash、外部觀察得到、不用做同步。缺點:I/O 成本;只適合低頻狀態。

## 設計理由

### 為什麼完全禁掉頂層 `let`

三個原因:

1. **舊行為是個地雷**。靜默丟掉 + gcc 錯誤是最糟糕的結果 —— 新手會困惑、
   而且難 debug。
2. **C interop 更乾淨**。每個 `.zy` 函式對應一個 C 函式;每個 `struct`
   對應一個 C struct;每個 `import` 對應一份原始碼前置。要把頂層 `let`
   乾淨地降階下來,就得發明 Zyen 端的 `static` 初始化順序去鏡像 C 的那一套
   (大家寫 shared library 都被它咬過)。
3. **上面四個 pattern 已經涵蓋所有真正用途**。我們有 2500 行的 app 跟
   40+ 模組的 stdlib 都遵守這套規矩,沒問題。

### 為什麼用零參數 `fn` 而不是 `const` 關鍵字

我們**本來可以**加檔案層級 `const fn` 或 `const name = value;`。沒加的原因:

- ZyenLang 反正每個 `fn` 都會 emit C prototype。把常數視為零參數 `fn` 是免費的 ——
  不需要新的降階路徑。
- 呼叫端 `max_pwm()` 讀起來像個值;括號只是小雜訊。
- 常數不需要 early-bind。如果值有天要根據平台不同而變,函式 body 是
  最自然的分支位置。

### 為什麼不允許檔案層級 `struct singleton = StructName { ... };`

理由跟上面 #2 一樣:會需要靜態初始化排序機制,我們沒有,也不需要。把
singleton 在 `main` 裡實例化、往下傳。資料流清楚;代價就是每個呼叫多一個
參數。

## 向後相容性

v0.1.49+ 無破壞性變更。v0.1.49 之前的行為是「靜默丟掉」這個 bug;v0.1.49
編譯器現在改成給清楚錯誤訊息。任何**真的**依賴靜默丟掉的使用者程式碼本來
就是壞掉的,新錯誤訊息嚴格優於舊的 gcc undeclared 訊息。

## 參考實作

- 編譯器:`zyenlang/transpiler.py` —— 頂層語句分類器拒絕 `let` / `const`
  並發出診斷訊息。
- Pattern 1 & 2 實例:`apps/zyide_gui.zy` 用零參數 `fn` 表達主題色
  (`col_bg()`、`col_panel()`、…)與尺寸(`win_w()`、`header_h()`、
  `body_y0()`、…)。
- Pattern 3 實例:`apps/zyide_gui.zy` 把 ~25 個狀態值透過每個 `redraw(...)`
  呼叫往下傳。是冗長,但是顯式。
- Pattern 4 實例:`zyenlang/std/log.zy` 把 logger 設定存在 disk 上跨呼叫共用。

## 待釐清問題

- 是否該在 `fn` body **內**加 `const` 關鍵字(目前 body 內有
  `const x: T = v;`,但實際上等同 `let` + 禁 `set`)。本 ZEP 範圍外。
- `import "..." as alias;` 是否該把 `struct` 定義以 qualified 形式
  (`alias.Type`)帶進來,而不是 global(`Type`)。目前被 import 進來的
  user struct 是 global、alias 在型別位置會被 strip 掉 —— 見
  `_strip_module_prefix_type` helper。未來可能有新 ZEP 重訪這條。

## 參考

- ZEP-0003(風格指南)—— 常數 fn pattern 的命名。
- C:`static` 儲存類別與初始化順序問題 —— 本 ZEP 兩個都避開了。
- Python:模組層級 `_CONSTANT` 慣例 —— ZyenLang 的 `max_pwm()` 是一樣的想法,
  寫成 call。

## 版權

本文件置於公有領域。
