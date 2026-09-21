# ZEP-0022：原生相容層與 C ABI v2

[English](ZEP-0022-native-compatibility-abi.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0022 |
| **標題** | 原生相容層與 C ABI v2 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-09-18 |
| **取代** | [ZEP-0019](ZEP-0019-native-modules-portable-toolchain.zh-TW.md) |
| **依賴** | [ZEP-0020](ZEP-0020-v0.3-core-language.zh-TW.md)、[ZEP-0021](ZEP-0021-projects-and-artifacts.zh-TW.md) |
| **參考實作** | `Ryan-2013/zyenlang` 的 `v0.3.0` tag |

## 摘要

ZyenLang 編譯成 C11，因此 C 互通是一個有型別的 build-time boundary，不是另一套
獨立的 runtime FFI。本 ZEP 定義一般 native declaration、宣告式 `.zlcm.h`
相容模板、callback/string ABI、toolchain metadata，以及供其他 build system
使用的 C/C++ 產物。

標準庫 native module 與第三方 package 使用完全相同的公開機制；標準庫沒有私有
C loading 語法。

## 直接 native declaration

Module 可宣告 package 內的 C input 與 typed symbol：

```zy
native source "bridge.c"
native link windows "user32"
private native fn open_native(path: str) i32 = "bridge_open"
```

Native path 相對宣告它的 module，且不可逃出 package root。source declaration
只接受 C source，不可指向任意 executable 或 install script。native symbol 會按
ZyenLang declaration 檢查，但 link/runtime 時仍是受信任 C code。

## Compiler-native 相容模板

Source 介面：

```zy
import std::c_module as c_module

let math: c_module::Module = c_module::load("native/math.zlcm.h")
let answer: i32 = math.add(20, 22)
```

`c_module::Module` 是 dependent compile-time type。型別與 initializer alias 必須
指向同一個 `std::c_module` import。load 只能直接初始化 local 或 struct/class
field default。模板 path 必須是相對目前 `.zy` 檔案的 string literal，且留在
package 內。裸 module type 不能當 parameter/return，因為缺少 initializer 決定
具體模板。

load 不產生 runtime call，也不啟動 Python wrapper。module graph loader 解析模板，
建立以 path hash 命名的隱藏 wrapper，再把通過驗證的 source、header、include/lib
path、library 與 flags 交給 artifact builder。同一路徑重用型別與 metadata；不同
路徑即使 `ZLC_MODULE` 同名仍是不同型別。一般 diagnostic 不可洩漏隱藏名稱。

已移除的 `import c_module.load("...") as name` 必須回報 migration error，並顯示
新 import 與直接宣告形式。

## 模板 macro

```text
ZLC_MODULE(name)
ZLC_HEADER("relative/header.h")
ZLC_SOURCE("relative/source.c")
ZLC_INCLUDE_DIR("relative/include")
ZLC_LIB_DIR("relative/lib")
ZLC_LIB("library")
ZLC_CFLAG("one-argument")
ZLC_LDFLAG("one-argument")

ZLC_STRUCT(Name, ZLC_FIELD(field, Type), ...)
ZLC_FN(zy_name, c_symbol, ReturnType, ZLC_PARAM(name, Type), ...)
```

Path/build macro 可加 `_WINDOWS`、`_LINUX`、`_MACOS`、`_UNIX`；`_UNIX`
適用 Linux、macOS 與其他 Unix。每個 CFLAG/LDFLAG macro 只能含一個 argument；
會改變輸出或 response-file 的 flag 必須拒絕。toolchain 一律接收 argument vector，
不得把模板字串拼成 shell command。

模板型別支援固定寬度數字、`usize`、`f32/f64`、`bool`、`str`、只作回傳的
`void`、`fn(P...) R` 與已宣告 `ZLC_STRUCT`。`int`、`float`、`ZL_String`
分別是 `i32`、`f64`、`str` 相容別名。0.3 不把 `List<T>` 與 raw pointer 當穩定
ABI 型別；native resource 必須藏在函式或 opaque handle class 後。

## C ABI v2

`zyenlang_c_abi.h` 定義公開 value boundary。`ZL_String` 是 immutable UTF-8
value，提供 retain、release、data、byte length。經 export function 回傳的
ZyenLang string 依生成 header contract 移交 owned result。

所有第一級函式值使用：

```c
typedef struct ZL_Function {
    void* call;
    void* env;
    ZL_ArcControl* owner;
    const char* signature;
} ZL_Function;
```

ABI 提供 `zl_fn_retain`、`zl_fn_release`、`zl_fn_assign`、`zl_fn_clear`、
`zl_fn_is_none`、`zl_fn_matches`、`ZL_FN_CALL_AS`。callback parameter 在 native
call 期間是 borrowed。C 若保存 callback，必須用 `zl_fn_assign` retain，並用對應
helper clear/replace。回傳 callback 需帶 owned reference。signature 驗證使用精確
canonical function type。

raw C callback API，包括沒有 `user_data` 的 API，由 C compatibility source
轉接；編譯器不會為任意 library 自動產生專用 trampoline。

## Toolchain 與產物

Native build 依序選擇：

1. 明確設定的 `ZY_CC`；
2. 內附 `toolchain/zig[.exe] cc`；
3. 系統 GCC、Clang 或 `cc`。

`ZY_CFLAGS`、`ZY_LDFLAGS` 可加入明確 build argument。`zy check` 會解析並驗證
native metadata，但不呼叫 C compiler 或執行 C。`zy build`、`zy run` 才會把
native source 與生成 C 一起編譯，因此跨入 trusted-code boundary。

`c-source` target 產生 generated source、含 `extern "C"` 的 C11/C++ header、
runtime/ABI header、宣告的 native source/local header，以及描述 source、header、
include/lib path、flag、library、export 的 metadata。static/shared target 編譯同一組
input。

## 原生 GUI 所有權

`std::gui::Application` 是原生視窗 session 的公開 owner。Retained widget 必須
透過該 instance 建立，例如 `app.label(...)`、`app.button(...)` 與
`app.column(...)`。每個 widget 都會 retain 自己的 Application，`draw()` 只會
繪製到該 Application。不存在 `gui::label(...)` 這類脫離 owner 的 module factory。

Raw drawing operation 同樣是 Application method。Application 關閉後再透過其
widget 繪製會回傳錯誤，不會改用隱含的 process-global target。目前的 Raylib
相容層每個 process 只允許一個已開啟的 Application，第二個同時 open 會被拒絕。
最後一個 Application reference 的 ARC cleanup 會關閉仍開啟的 native session。

Source API 必須保持 backend-neutral，不暴露 Raylib handle。Raylib 是初始相容
實作，不是 source-level contract。下一個 native backend 目標為 SDL3 的 window、
event、text input/IME 與 audio，加上 SDL_GPU rendering。可選的 Dear ImGui 工具可
位於此 boundary 後方；application code 繼續使用 retained Application/widget API。

## 驗證與信任

編譯器拒絕未知或 malformed macro、重複名稱、非法 C identifier、不支援或遞迴
ABI value、衝突 symbol signature、缺少檔案、package escape、非法 library name
與不安全 flag。這些檢查保護 compiler command structure 與型別一致性。

它們不會 sandbox C。native source 可用應用程式權限存取記憶體、檔案、網路與
process API，dependency 仍需人工審查。依舊 ABI layout 預編譯且按值傳遞的 wrapper
必須為 ABI v2 從 source 重建。

`v0.3.0` 本輪只交付 source tag；portable archive、MSI 與 GitHub Release asset
屬於另一輪平台驗證與發行工作。
