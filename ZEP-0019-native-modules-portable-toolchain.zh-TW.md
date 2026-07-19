# ZEP-0019 - 原生模組與可攜式 C 工具鏈

[English](ZEP-0019-native-modules-portable-toolchain.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0019 |
| **標題** | 原生模組與可攜式 C 工具鏈 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Draft |
| **類型** | Standards |
| **建立日期** | 2026-07-19 |
| **相關規格** | [ZEP-0007](ZEP-0007-zy-doctor.zh-TW.md)、[ZEP-0017](ZEP-0017-package-manager.zh-TW.md) |

## 摘要

ZyenLang 會轉譯成 C，因此原生互通是正常建置流程的一部分，不是另一套外部
執行環境。本 ZEP 規範 C 編譯器選擇、可攜式發行版內容、`std/c_module` 的
編譯期行為、內附的 `std/tk` GUI 後端，以及原生套件的信任邊界。

可攜式發行版內建 Zig C 編譯器與該平台的 GUI runtime。使用者解壓縮並把
命令目錄加入 `PATH` 後即可使用，不必另外安裝 Python、GCC 或 Raylib 開發套件。

## C 編譯器選擇

會產生或執行原生執行檔的命令，依下列順序選擇編譯器：

1. 若有設定 `ZY_CC`，使用它指定的命令。
2. 使用 ZyenLang `toolchain/` 目錄內附的 Zig，並以 `zig cc` 呼叫。
3. 依序尋找系統上的 `gcc`、`clang`、`cc`。

選擇第一個可用的候選者。`zy doctor` 必須使用相同的解析器，並回報編譯器
種類以及它是內附或系統提供。只要內附的 Zig cc 可用，就不能因為系統沒有
`gcc` 而回報缺少編譯器。

Zig cc 在這裡只作為 C 後端。ZyenLang 不會被轉換成 Zig，也不依賴 Zig 語言
或 Zig 套件管理器。

## 原生建置流程

`c_module.load()` 是編譯期宣告。它不會產生執行期載入呼叫，也不會啟動 Python
wrapper 產生器。

執行 `zy run` 或建置執行檔時，編譯器依序進行：

1. 解析 `.zy` import，以及每個字串常值指定的 `.zlcm.h` 或 `.zlcm.json`。
2. 為每個不同的已解析模板產生內部 ZyenLang wrapper 型別。
3. 把展開後的 ZyenLang 程式轉譯成 C。
4. 在同一次編譯與連結命令中，傳入生成的 C 和所有 `ZLC_SOURCE` 原始碼。
5. 強制 include 宣告的 header，加入 include、library 目錄，再套用目前平台的
   library 與 flags。
6. 連結成一個原生執行檔。

Header 只會被 include，不會被當成獨立 translation unit 編譯。`ZLC_LIB` 宣告
的預編譯函式庫會在最後階段連結。

`zy check` 會解析模板、展開 wrapper 並檢查呼叫型別，但不會啟動 C 編譯器。
只輸出 C 的 build 會產生 `.c`，不會編譯原生來源。因此 C 編譯器診斷只會在
產生執行檔時出現。

## 可攜式發行版內容

每個支援平台都有獨立的可攜式壓縮檔，內容包含：

- `zy` 命令、編譯器 runtime 與標準庫；
- 可作為 Zig cc 使用的內附 Zig 工具鏈；
- 公開的 `zyenlang_c_abi.h`；
- `std/tk` 的 ZyenLang facade 與 C 相容層原始碼；
- `std/tk` 使用的該平台 Raylib 動態函式庫。

GUI 動態函式庫在 Windows 是 `raylib.dll`，Linux 是 `libraylib.so`，macOS 是
`libraylib.dylib`。`std/tk` 可以從 `ZYENLANG_RAYLIB`、可攜式發行版、執行檔
目錄或平台函式庫搜尋路徑找到它。

「可攜式」代表不需要另外安裝編譯器、Python 或 Raylib 開發套件，但仍依賴
作業系統 ABI、視窗系統、顯示驅動及 C runtime。一個作業系統或 CPU 架構的
發行版不能直接當成另一個平台的發行版。

`std/tk` 使用公開的 `std/c_module` 機制。標準庫原生模組不享有私有 import
語法或只有編譯器能使用的載入特權。它受到信任，是因為它隨經過簽章或雜湊
驗證的 ZyenLang 發行版一起交付及測試。

## 型別名稱與撞名防護

原始碼宣告仍使用公開型別 `c_module.Module`。編譯器根據模板的 canonical path
與穩定的 SHA-256 路徑雜湊產生具體隱藏 wrapper 型別。因此：

- 同一個已解析模板載入兩次時，共用一個型別與一份 metadata；
- 不同路徑即使使用相同 `ZLC_MODULE` 名稱，仍是不同型別；
- 一般使用者診斷不顯示隱藏名稱。

從模板讀取的 module、function、struct、field 與 parameter 名稱，都必須先
驗證，才能生成原始碼。名稱必須是普通的 C／ZyenLang identifier，不能包含
空白、控制字元、換行、註解、標點或原始碼片段。編譯器必須維護 entry point、
ABI runtime 與生成用內部符號的保留名稱集合。

路徑雜湊能避免 wrapper 意外撞名，但它不是安全沙箱，也不能讓任意原生 C
自動變安全。

## 原生程式碼信任邊界

`.zlcm.h` 模板及其選擇的每個 `.h`、`.c`、函式庫、編譯器 flag 與 linker flag
都是原生程式碼輸入。原生模組可以使用作業系統 API、讀寫目前 process 能存取
的檔案、連線網路、建立 thread 或終止程式。C constructor 也可能在產生的
執行檔啟動時、ZyenLang `main` 之前執行。

因此：

- 本機原生模組的信任等級，等同直接加入應用程式的 C 原始碼；
- 下載套件時不得執行套件，但建置或執行 native 套件就會跨越原生信任邊界；
- registry 套件必須以 `zy.lock` 記錄的精確 SHA-256 驗證；
- registry 套件不能透過絕對路徑、`..`、symlink、source path、include path 或
  library path 逃出不可變 package root；
- registry 的 compiler 與 linker flags 使用保守白名單；
- 超出政策的 flag 或外部路徑需要明確的本機 unsafe-native 授權，且不得發布為
  portable 套件；
- 編譯命令必須以參數陣列傳入，不能串成 shell command；
- 第三方 native 套件第一次加入時，必須取得明確的原生程式碼確認，除非已由
  政策或 lockfile 狀態信任。

鎖定與雜湊只能證明選到的是哪些 bytes，不能證明那些 bytes 沒有惡意行為。
文件不得把 native 套件描述為已沙箱化。

## 目前實作狀態

| 要求 | 狀態 |
|---|---|
| 優先使用內附 Zig cc，再尋找系統編譯器 | 已實作 |
| 生成 C 與 `ZLC_SOURCE` 一起編譯及連結 | 已實作 |
| `c_module.load()` 在編譯期消失 | 已實作 |
| 隱藏 wrapper 型別使用路徑雜湊，metadata 去重 | 已實作 |
| 跨平台 `std/tk` C 相容層 | 已實作 |
| 每個可攜式發行版內附 Raylib runtime | 已實作 |
| 嚴格驗證模板提供的所有符號 | 部分實作 |
| package root 限制、flag 政策與 native 確認 | 等待 ZEP-0017 實作 |

安全相關項目完成並通過惡意輸入測試以前，本 ZEP 維持 `Draft`。

## 必要測試

- 系統 `PATH` 沒有 C 編譯器時，仍可用可攜式發行版 build 與 run 基本程式。
- 只使用發行版內附 GUI runtime 開啟 `std/tk` 視窗。
- 驗證 `ZY_CC`、內附 Zig cc、GCC、Clang 與沒有可用編譯器時的選擇結果。
- 驗證不同路徑的同名模組不撞名，同一模板只生成一次。
- 在啟動 C 編譯器前，拒絕控制字元、原始碼片段、保留符號、路徑逃逸、危險
  flags 與 symlink 逃逸。
- 驗證 `zy check` 絕不編譯或執行原生來源。
- Windows、Linux、macOS 的可攜式發行版必須分別測試。

## 版權

本文件置於公有領域。
