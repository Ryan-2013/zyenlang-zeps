# ZEP-0005 — 錯誤處理慣例

[English](ZEP-0005-error-handling.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0005 |
| **Title** | Error Handling Conventions |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Informational |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

ZyenLang **沒有 exception**。錯誤就是值:`int` 回傳碼、`None` 的 `ptr<T>`、
或空的 `List`。本 ZEP 定義哪種錯誤該用哪種形狀,以及標準函式庫的現行做法。

## 動機

當一個語言有多種方式表達失敗,呼叫者就會處理一部分而漏掉另一部分。編譯器
幫不上忙:沒有 checked exception、沒有 `Result<T, E>` 能 pattern-match。所以
慣例必須撐起這件事。

stdlib 大致上已經遵守下列規則;本 ZEP 把它寫清楚,以免新模組漂移。

## 規範

### 三種錯誤形狀

| 形狀 | 何時用 |
|---|---|
| **`int` 回傳碼** | 函式是動作(write、mkdir、kill)。`0` 是成功,非零是錯誤類別。 |
| **`ptr<T>` `None`** | 函式回傳可能存在的值。`None` = 不存在或失敗;有效 `ptr<T>` = 存在。 |
| **空 `List`** | 函式回傳集合。空 = 沒有結果(無論是因為失敗還是本來就空 —— 若操作是冪等的,這個區分不重要)。 |

### 怎麼選

```text
   函式成功時回傳什麼?
   │
   ├── 沒有有意義回傳(動作)  ──→  int 回傳碼
   │
   ├── 一個 T 型別的值          ──→  ptr<T> + None 表不存在
   │                                  或拆成兩個函式,當 T 是 int / str /
   │                                  bool 之類 primitive 時
   │
   └── 多個 T 型別的值          ──→  List
                                      (呼叫端檢查 is_empty)
```

「兩個函式拆分」是 escape hatch,用在 `ptr<str>` 對一個本質上是字串的東西
顯得太重時。stdlib 在 `fs.read` 用這招(檔案不存在時回 `""`;呼叫端在意
的話自己先 `fs.exists`)。

### int 回傳碼慣例

回傳 `int` 的函式:

- **`0`** —— 成功。
- **負值** —— 分類錯誤。stdlib 用小的負數(`-1`、`-2`、`-3`),
  函式 contract 文件化每個負值意義。
- **正值** —— 保留給下游工具的 exit code(例如 `cmd.run` 把子 process
  exit status 原樣回傳,大多失敗情況是正值)。

`std/mem` 的例子:

```zy
fn free(p: ptr<Any>) -> int {
    return zl_mem_free(p);
    // 回傳:0=ok、-1=null、-2=not-owned、-3=already-freed
}
```

### `ptr<T>` None 慣例

None pointer 是 `T` 型值「不存在」的唯一認可 sentinel。呼叫者必須先檢查:

```zy
let p: ptr<Config> = config.load_optional("settings.ini");
if (!mem.is_valid(p)) {
    set p = mem.alloc(default_config());
}
let cfg: Config = *p;
```

`mem.is_valid(p)` 是標準的 None 檢查。**不要**靠 `ptr` 的 truthiness ——
ZyenLang 沒有 `if (p)` 簡寫,而且 None sentinel 在 C 端不一定是數值 0。

### Panic 是給 runtime invariant 用的

有些操作真的沒辦法有意義地回傳錯誤:

- `List.get(out_of_range_index)` —— 沒有 `Any` 型別的 fallback 值。
- 對 `mem.is_valid` 已經 catch 的 None `ptr<T>` 再 deref。
- `std/buffer` 內 buffer overflow。

這些會立刻 abort process 並把診斷訊息寫到 stderr。**沒有 `try` / `catch`
能恢復**。**不要設計**會在輸入錯誤時 panic 的新 API;panic 只留給呼叫者
事前無從預防的 runtime invariant 違反。

如果一個函式可能收到使用者來的壞輸入,就回 `int` 碼或 `None`。如果一個
函式只被信任的內部程式呼叫,可以 panic。

## 反模式

### 不要用 sentinel 值偷藏錯誤

```zy
// NO —— -1 表錯誤,但呼叫端沒讀註解看不出來。
fn lookup_age(name: str) -> int {
    // 不存在時回 -1
    ...
}

// YES —— 回 ptr<int>,不存在情況顯式。
fn lookup_age(name: str) -> ptr<int> {
    ...
}
```

例外:當值的型別**本身**已經排除有效 sentinel(例如本質為正的量用 `-1`
表示不存在),拆成兩個函式(`lookup_age` + `has_age`)比混 ptr 跟 int 乾淨。

### 不要對有更豐富資訊的動作回 `bool`

```zy
// 不夠好用
fn write_file(path: str, body: str) -> bool { ... }

// 更好 —— int 碼讓呼叫端分得清「目錄不存在」和「權限不足」
fn write_file(path: str, body: str) -> int { ... }
```

### 不要靜默吞掉錯誤

```zy
// NO
fn save(path: str, data: str) -> void {
    fs.write(path, data);   // 回傳碼被忽略
}

// YES
fn save(path: str, data: str) -> int {
    return fs.write(path, data);
}
```

即使直接呼叫者不在意,把碼往上傳就保留了「呼叫者的呼叫者」決定的空間。

## 設計理由

### 為什麼沒有 exception

- ZyenLang 降階到 C。實作 exception 不是 setjmp / longjmp(慢、資源
  cleanup 脆弱)就是 stack-walking unwinder(runtime 體積大)。
- Exception 不可見地穿過函式邊界。呼叫者會忘記處理。靜態分析難搞。
- 上面三種形狀已經乾淨地涵蓋 exception 的使用場景。我們可以不用 exception
  寫完整個 stdlib,讀起來大致一樣乾淨。

### 為什麼沒有 `Result<T, E>`

要加超出現有 `ptr<T>` 和 `List` 的 generic 型別,等同重寫 type checker。
兩種被認可的形狀(`ptr<T>` + None、`int` 碼)用無需新機制就解決同樣的問題。

### 為什麼區分「動作」和「optional 值」

務實考量,不是哲學:這兩種在呼叫端的流動方式不同。動作傾向鏈式(做這個、
然後那個);optional 傾向分支(有就用、沒有就 fallback)。挑對形狀讓
call site 好讀。

## 參考實作

- `std/fs` 的 `write`、`mkdir`、`remove` 回 `int` —— 動作。
- `std/mem` 的 `alloc` 回 `ptr<T>`、`free` 回 `int` —— optional vs 動作。
- `std/list` 的 `index_of_str` 回 `int`,`-1` 表 not-found。這是「不要偷藏
  sentinel」的紀錄例外:回傳型別是個**位置**,`-1` 是通用的「不存在位置」
  慣例。
- `std/log` 的 `info` / `warn` / `error` 回 `int`。

## 待釐清問題

- 是否該標準化一個 `Error` struct(含 `code: int`、`message: str`、
  `cause: ptr<Error>`)做更豐富的回報。本 ZEP 範圍外;會是另外一份
  Standards 提案。
- `cmd.run` 是否該把 stderr 跟 exit code 分開。目前只回 exit code;
  stderr 流向 parent 的 stderr。

## 參考

- ZEP-0004(全域狀態)—— 錯誤回報常與共享狀態(如 `errno` 風格)互動;
  我們把碼放在回傳值裡,避開這條路。
- Go 的 `(T, error)` 回傳慣例 —— ZyenLang 的拆函式是相關形狀,
  只是沒語法糖。

## 版權

本文件置於公有領域。
