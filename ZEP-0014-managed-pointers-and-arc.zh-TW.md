# ZEP-0014 - 受管指標與自動參考計數

[English](ZEP-0014-managed-pointers-and-arc.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0014 |
| **標題** | 受管指標與自動參考計數 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-07-18 |
| **依賴** | [ZEP-0010](ZEP-0010-first-class-functions.zh-TW.md)、[ZEP-0013](ZEP-0013-closures.zh-TW.md) |

## 摘要

ZyenLang 保留明確的指標語法，但語言層的 `ptr<T>` 是會檢查狀態的受管
handle。Owned pointer 與 closure environment 由編譯器自動插入 atomic ARC
操作；為了 C 互通與區域變數取址，語言仍允許 borrowed pointer。

本 ZEP 規範巢狀指標型別、owned 配置、解參考、轉型、alias、函式回傳時的
所有權轉移、struct 的 managed field，以及此安全模型的邊界。

## 指標種類

### 普通區域儲存

```zy
let value: int = 10;
```

普通區域變數使用 automatic storage。宣告本身不代表 heap 配置；函式作用域
結束時，它的語意生命週期也結束。

### Borrowed pointer

```zy
let value: int = 10;
let borrowed: ptr<int> = &value;
```

`&value` 建立 owner 為 null 的 `ptr<int>`，不會延長 `value` 的生命週期。
編譯器會拒絕明顯回傳 stack pointer 的程式，但 ZyenLang 並不是完整的 borrow
checker。

### Owned pointer

```zy
let *owned: ptr<int> = 10;
```

`let *` 會配置 owned cell、用右側值初始化，並把受管 `ptr<int>` handle 放進
`owned`。複製時 retain 相同 owner；離開作用域時 release 一個 reference。

## 遞迴指標型別

`ptr<T>` 可以遞迴，pointee 本身也可以是 pointer：

```zy
let *two: ptr<ptr<int>> = 10;
let *three: ptr<ptr<ptr<int>>> = 42;
print(**two);
print(***three);
```

每個受管層級擁有的是一個 `ZL_ptr` 值，不是 raw C pointer。因此
`ptr<ptr<int>>` 的 ABI 不等同 C `int **`。相容層要轉接 raw
pointer-to-pointer API 時，必須使用原生 address 欄位。

當 owned declaration 的初始化式足以推導內層型別時，可以寫 `ptr<ptr>`；
公開 API 仍建議寫完整型別。

## 解參考與寫入

每一個 unary `*` 只移除一層 `ptr`。對 `None`、已 free 的 owner 或 runtime
type 不相容的值解參考時，會得到明確 runtime error，不會直接進入無效 C 位址。

```zy
let *p: ptr<ptr<int>> = 10;
set **p = 20;
print(**p);
```

`ptr<void>` 不可直接解參考，必須先明確 cast 成具體 pointer 型別。

## 受管 address identity

對 managed pointer 而言，`&*p` 會得到 `p` 的 alias，保留同一個 address、ARC
owner、ownership state 與 runtime type tag。這個規則可以連續套用：

```zy
let *source: ptr<ptr<int>> = 10;
let *slot: ptr<ptr> = &**source;
set **slot = 21;
print(**source); // 21
```

Alias 會 retain owner，因此即使外層 chain 離開 scope 仍保持有效。這是受管
identity 操作，不是暴露一個沒有追蹤的 raw interior address。

## 轉型與 `ptr<void>`

```zy
let opaque: ptr<void> = typed;
let restored: ptr<int> = (ptr<int>)opaque;
```

- `ptr<T>` 可隱式轉成 `ptr<void>`。
- `ptr<void>` 回到 `ptr<T>` 必須明確 cast。
- 不同具體 pointer 型別互轉時必須明確 cast。
- Cast 不會改變 address、owner、ownership state 或 runtime type tag。
- 函式值不可與 `ptr<T>` 互相 cast。

## 回傳與逃逸語意

函式回傳 managed value 時，會把一個 owned reference 轉移給呼叫端。清理
local 前，編譯器先 retain 回傳值；接著正常 scope cleanup 只釋放 local
reference。

```zy
fn make_value() -> ptr<int> {
    let *value: ptr<int> = 110;
    return value;
}
```

區域變數名稱不影響語意；每次呼叫都有獨立 storage 與 ARC reference。同一
規則會遞迴套用到 nested pointer 與回傳 struct 裡的 managed field。

```zy
struct Box {
    let this.value: ptr<int>;
}

fn make_box() -> Box {
    let *value: ptr<int> = 110;
    return Box { value: value };
}
```

從回傳值可到達的 managed values 都會被保留；無法從回傳值到達的 managed
locals 仍在 scope exit 時釋放。

## 函式值與 closure

函式值使用 ABI v2 `ZL_Function { call, env, owner, signature }`。具名 top-level
function 沒有 owner。Closure environment 使用 atomic ARC；destructor 會遞迴
釋放捕獲的函式值、owned pointer 與 managed struct field。

函式參數在呼叫期間是 borrowed；回傳 managed value 時轉移一個 owned
reference。這些規則也適用於跨越 `c_module` 相容層的值。

## 明確 free

`mem.free(p)` 會立即讓所有 alias 看到 owned payload 已失效：

- `0`：成功；
- `-1`：`None`；
- `-2`：borrowed pointer；
- `-3`：已經 free。

ARC control block 會保留到最後一個 alias 被釋放。除非需要提早失效，否則應
優先使用自動 ARC cleanup。

## 安全邊界

- ARC 不會回收 reference cycle。
- Borrowed pointer 仍受來源生命週期限制。
- Managed pointer 不允許 pointer arithmetic。
- Managed pointer 目前沒有用於索引的 allocation bounds。
- 外部 C library 若要跨呼叫保存值，必須使用 ABI v2 retain/release 與 pointer
  adoption helpers。

## 參考實作

參考實作為 ZyenLang v0.1.53。回歸測試位於
`tests/pointer_escape_test.zy`、`tests/nested_ptr_init_test.zy`、pointer/memory
測試、closure 測試與 c_module callback 測試。

## 版權

本文件以 CC0-1.0 釋出至公有領域。
