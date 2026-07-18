# ZEP-0016 - List 結構式方法派發

[English](ZEP-0016-structural-list-dispatch.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0016 |
| **標題** | List 結構式方法派發 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-07-18 |
| **依賴** | [ZEP-0011](ZEP-0011-struct-body-methods.zh-TW.md)、[ZEP-0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md) |
| **參考實作** | [ZyenLang v0.1.83](https://github.com/Ryan-2013/zyenlang/releases/tag/v0.1.83) |

## 摘要

本 ZEP 允許異質 List 保存使用者定義的 struct，並直接呼叫共同方法；語言
不必新增 interface、trait、繼承、公開 union 或公開 `Any` 型別。相容性由
方法名稱與完整函式簽章以結構方式決定。

## 語法

```zy
struct Dog {
    fn bark() -> void {
        print("dog");
    }
}

struct Cat {
    fn bark() -> void {
        print("cat");
    }
}

fn main() -> int {
    let animals: List = [Dog {}, Cat {}];
    for (let i = 0; i < animals.len(); set i += 1) {
        animals.get(i).bark();
    }
    return 0;
}
```

## 靜態相容規則

編譯器會保守記錄具名區域 List 在 literal 與直接 `append`、`append_ptr`、
`set` 中出現的元素型別。只有在每個可能元素都是使用者 struct，而且每個
struct 都提供相同方法名稱、參數型別、回傳型別與預設值表達式時，結構式
呼叫才會通過。

結構式呼叫使用位置參數；參數名稱不屬於共同形狀。只要靜態型別集合仍存在，
缺少方法、混入 scalar 或簽章不同都會在編譯期報錯。

## 擦除 List 邊界

一般 `List` 參數會擦除封閉元素集合，因為被呼叫函式可能修改 List。擦除後
只有在整個程式對該方法名稱存在唯一且不含糊的簽章時才能編譯。產生的程式會
在派發前檢查 boxed runtime type；不相容值會產生 runtime error，而不是
呼叫錯誤的 C 函式。

## 儲存與所有權

ABI v3 在內部 `Any` 加入 boxed struct variant。append struct 時會複製值到
ARC-owned box，並遞迴 retain 其中的 managed 欄位。

- `get()` 回傳 List cell 的 borrowed view。
- 把該 view 複製到另一個 List 時會 retain box。
- `set()` 與 `clear()` 會 release 被移除的 box。
- `pop()` 會轉移 box 引用。
- `(StructType)value` 會先檢查精確 runtime type 再還原值。

方法 receiver 指向 box 內的值，所以會修改 receiver 的方法會直接更新 List
元素。用來初始化 List 的原始 struct 仍是另一份獨立的值拷貝。

## 邊界

外部 c_module struct 不會直接裝箱；應以使用者定義的 ZyenLang facade struct
建立語言邊界的 ownership 與方法。List 仍不接受裸 `fn(...)` 值；需要在
List 保存 callback 時，可將函式放入使用者 struct 欄位。

List 本身尚未由 ARC 管理，ARC 也不收集循環引用。需要確定釋放 managed
元素時，程式應呼叫 `clear()`。
