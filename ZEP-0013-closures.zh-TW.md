# ZEP-0013 — 閉包與 Lambda Lifting

[English](ZEP-0013-closures.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0013 |
| **標題** | 閉包與 Lambda Lifting |
| **作者** | zuenchen、Claude Opus 4.7 |
| **狀態** | Active |
| **類型** | Standards |
| **建立日期** | 2026-06-21 |
| **修訂歷史** | 2026-06-21 |
| **取代** | [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md) 的「禁止巢狀 fn」條款 |

## 摘要

ZyenLang 允許在 `fn` body 裡面再定義一個 `fn`。內層 fn 在構造時
依值捕獲外層 scope 的識別字,然後可以被當成 `fn(...)->T` 值回傳或
儲存。編譯器把內層 fn 提升(lambda-lift)到檔案範圍,並把捕獲的
狀態包進 heap 配置的環境記錄。**閉包內部對捕獲值的存取是唯讀的**。

## 動機

[ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md) 把一等公民
函式值放出來了,但明確禁止巢狀 fn 定義,理由是閉包需要 heap 環境
與生命週期追蹤。使用者接受了這條規則一輪,馬上撞到 ZEP-0010 無法
表達的明顯模式:

```zy
fn make_repeater(f: fn(int,int,int)->int, rounds: int) -> fn(int)->int {
    fn back_fn(k: int) -> int {
        for (let i = 0; i <= rounds; set i += 1) {
            f(i, i, k);
        }
        return 0;
    }
    return back_fn;
}
```

struct 欄位的替代方案(`Repeater { f, rounds }` 加上 `.run(k)`
方法)能跑,但把樣板程式推到**呼叫端**而不是工廠端。對於要產生很多
小閉包的工廠,「每個閉包一個 struct」太冗長。

本 ZEP 把缺失的直接寫法補回去,代價是放棄「不配置 heap」的原始
立場 —— 但成本被嚴格限制:**只有閉包**會配置 heap,而且每次建立
閉包只 `malloc` 一次。

## 規範

### 語法

在另一個 `fn` body 裡面寫 `fn` 定義現在合法:

```zy
fn outer(<params>) -> <ret> {
    ...
    fn inner(<params>) -> <ret> {
        // body —— 可以引用 outer 的 params 和 inner 定義之前
        // 在 outer body 裡宣告過的 locals
    }
    ...
    return inner;       // 或 let x = inner; 等
}
```

內層 fn 的名字是外層 fn 的**區域 fat-value 符號**(不能用裸名從
其他地方呼叫)。在 outer body 內,內層 fn 定義那行之後,該名字
就是 `fn(<param-types>)-><ret>` 型別的值。

### 捕獲分析

編譯器計算內層 fn 的*自由變數*集合:body 裡用到、但

- 不是它的參數,
- 不是它 body 內用 `let` / `for (let ...)` 宣告的,
- 不是檔案範圍(頂層 fn、struct、const),
- 不是 ZyenLang 關鍵字或內建型別,
- **而且**在內層 fn 定義那行,在外層 fn 的 scope 內(外層 params
  + 較早宣告的 locals)。

符合條件的集合就是閉包的捕獲。

### 捕獲語意

捕獲是構造時刻的**唯讀快照**。具體:

1. 內層 fn 定義那行,編譯器 emit env struct 的 `malloc`(每個
   捕獲一個欄位),把每個捕獲的當下值複製進去,再組出 fat value
   `ZL_Function { call, env, owner, signature }`。
2. 每次呼叫閉包時,lifted fn 把 env 裡的捕獲複製回區域變數(讓
   body 可以用原本的名字引用)。
3. 在閉包 body 內寫 `set <capture> = ...` 只會改動該次呼叫的區域
   副本。env 和外層原本的綁定都不受影響。
4. 後續每次呼叫都會重新從 env 讀:看到的永遠是構造時的快照。

如果需要「閉包間共享、可變」的狀態,改用 struct 加 fn 型別欄位
(ZEP-0010 + [ZEP-0011](ZEP-0011-struct-body-methods.zh-TW.md)
的模式)—— struct 本身就是可變狀態的容器。

### 生命週期

Env 記錄依 [ZEP-0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md) 使用
atomic ARC。複製、指派、捕獲、儲存或回傳 closure 時 retain owner；離開
scope 或覆寫時 release。最後一次 release 會執行 environment destructor，
遞迴釋放 managed captures。

### 仍然禁止的構造

- **捕獲 `List`** —— 捕獲是依值;`List` 捕獲會複製 struct 但共享
  heap 緩衝,造成混淆的別名。編譯器目前不在 parse 階段拒絕,但
  行為屬於未定義。避免。
- **兩層巢狀** —— `fn outer() { fn middle() { fn inner() {...} } }`
  屬於未定義行為;實作不會走第二層。
- **改變捕獲的 fn 值** —— 同上;當作唯讀。
- **匿名函式字面值** —— 仍然沒有 `fn(x) {...}` 表達式語法。要用
  巢狀 fn 請給它名字。

## 設計理由

### 為什麼用 lambda lifting(而不是胖閉包)

Lambda lifting 把每個巢狀 fn 搬到檔案範圍當普通 C 函式。閉包值包含
lifted call pointer、environment、ARC owner 與 canonical signature。
替代方案(裡面藏 trampoline 的
胖閉包)需要 runtime 程式碼生成或 libffi,都不符合專案。Lambda
lifting 是教科書級的「編譯到 C」標準做法。

### 為什麼是唯讀捕獲

真正的讀寫捕獲需要共享 cell(Python 的 `cell` 型別、C++ 的
`std::shared_ptr<int>` 等),這樣多個閉包跟外層才能看到同一個
mutation。ZyenLang 的 ARC 管理每份唯讀 snapshot 的生命週期，但不改變
capture mutation 語意：每個 closure 仍擁有自己的 snapshot。

struct 欄位模式已經乾淨地處理了「可變共享」的情況,所以我們沒有
失去表達力。

### 為什麼使用自動 ARC

函式值會流經 local、struct field、capture、assignment 與 return。由編譯器
自動插入 ARC，可用同一套 ownership 規則涵蓋所有路徑，又不必在 ZyenLang
原始碼加入 retain/release 語法。Atomic control block 也允許 C callback
adapter 跨 thread 安全保存或釋放 closure ownership；captured data 本身不會
因此自動變成 thread-safe。

## 向後相容性

- 所有 ZEP-0010 fn-value 程式仍然編譯。具名 fn 引用現在會走生成的
  thunk + 常數 fat literal(`<name>_zlfnval`)，沒有 environment owner。Thunk
  丟掉 env 參數,直接 forward 給原本的具名 fn。
- 對 fn 型別區域使用 `f(args)` 的呼叫,現在編譯為 C 的
  signature-checked ABI helper。ZyenLang 表面語法不變。
- ZEP-0010 那個「巢狀 fn」錯誤不再觸發 —— 同樣的源碼現在會編譯
  成閉包。

## 參考實作

目前參考實作在 ZyenLang v0.1.53（`zyenlang/transpiler.py`）：

### 新增 / 變更的部件

| 部件 | 用途 |
|---|---|
| `emit_fn_typedefs` | 在共同 `ZL_Function` ABI 上產生 signature-specific checked-call helper |
| `emit_fn_thunks`(新) | 每個頂層 fn 一個 `<name>_zlthunk` + `<name>_zlfnval` |
| `coerce_named_fn_to_fnval`(新) | 期望型別為 fn 時,把裸 `<name>` 換成 `<name>_zlfnval` |
| `transform_function_calls` | fn 型別值的呼叫會通過 signature-specific checked helper |
| `transform_method_calls` | fn 型別 struct field 使用相同 checked call path |
| `scan_nested_fns`(新 pre-pass) | 走每個 fn body;對每個巢狀 fn,算捕獲、註冊 lifted fn 到 ctx |
| `emit_lifted_env_structs`(新) | 每個閉包一個 `typedef struct ZL_env_<outer>_<inner> { … };` |
| `emit_lifted_fn_prototypes`(新) | 每個 `<outer>_<inner>_zllifted` 的 forward declaration |
| `emit_lifted_fn_bodies`(新) | 在檔案末尾定義每個 lifted fn;先把捕獲複製進區域變數,再正常 transpile body |
| `emit_function_body` | 看到巢狀 fn 定義時配置 ARC env、retain managed captures，並建立 `ZL_Function` local |

### Codegen 順序

1. `scan_nested_fns` —— 註冊 lifted fn 型別。
2. `collect_fn_typedefs` —— 收集所有 fn 型別。
3. `emit_struct_forward_decls`。
4. `emit_fn_typedefs` —— signature-specific checked-call helpers。
5. `emit_struct_defs`。
6. `emit_lifted_env_structs`。
7. `emit_function_prototypes` —— 真實 fn。
8. `emit_lifted_fn_prototypes`。
9. `emit_fn_thunks` —— 具名 fn 的 thunk + fnval 常數。
10. (runtime helpers、std 模組 C stub 等)
11. 主 fn emit 迴圈。
12. `emit_lifted_fn_bodies` —— lifted 閉包 body 最後。

### 捕獲分析速查

- `inner_local_bindings` = 內層 params + body 裡出現在 `let `、
  `const `、`for (let ` 之後的識別字。
- `used` = body 裡所有識別字(字串字面值先剝掉)。
- `captures` = `used - inner_local_bindings - keywords - file_scope`,
  再過濾出在外層 fn 定義時 scope 中存在的。

### Lifted fn body 骨架

```c
static int <outer>_<inner>_zllifted(void* env_v, int k) {
    ZL_env_<outer>_<inner>* __zl_env = (ZL_env_<outer>_<inner>*)env_v;
    int rounds = __zl_env->rounds;            // 提升的捕獲
    ZL_Function f = __zl_env->f;              // 呼叫期間 borrowed snapshot
    /* 原本的巢狀 fn body,照常 transpile */
}
```

`tests/closure_test.zy`、`tests/fn_chain_test.zy` 與
`tests/c_module_callback_test.zy` 涵蓋 capture 語意、ARC ownership、立即鏈式
呼叫，以及由 C 相容層 retain 的 callback。

## 待解問題

- **可變捕獲(透過 cell)。** 加上 `cell<T>` 包裝(Python 風格)
  能讓閉包之間共享變動。目前參考實作範圍外。
- **閉包巢閉包。** 兩層巢狀需要 lifted 中層 fn 的 body 自己再做
  lambda lifting。`scan_nested_fns` 邏輯上可以遞迴,但目前未測試,
  視為未定義。
- **`List` 捕獲。** 要嘛 parse 階段拒絕,要嘛文件說明捕獲的
  `ZL_List` 是淺拷貝。
- **方法引用捕獲。** `let f = some_struct.method;` 仍不支援
  (ZEP-0010 的開放問題)。閉包不改變這條。

## 參考資料

- [ZEP-0003](ZEP-0003-style-guide.zh-TW.md) —— 表面語法克制原則
  (不要 lambda 字面值、不要 arrow 運算子)。
- [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md) —— 本 ZEP
  賴以延伸的 fn-value 基礎。ZEP-0010 的「禁止巢狀 fn」規則被取代;
  其他規則仍然有效。
- [ZEP-0011](ZEP-0011-struct-body-methods.zh-TW.md) —— struct 方法
  寫法,需要可變共享狀態時的推薦模式。

## 版權

本文件置於公有領域。
