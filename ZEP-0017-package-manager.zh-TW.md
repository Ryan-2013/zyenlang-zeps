# ZEP-0017 - `zy pkg` 套件管理器

[English](ZEP-0017-package-manager.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0017 |
| **標題** | `zy pkg` 套件管理器 |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Draft |
| **類型** | Standards |
| **建立日期** | 2026-07-19 |
| **依賴** | [ZEP-0008](ZEP-0008-zyenv.zh-TW.md) |

## 摘要

本 ZEP 定義一套像 `pip install` 一樣方便、但可重現的 ZyenLang 套件流程。
套件管理器使用 `zy pkg` 命令、專案 manifest、精確 lockfile 與不可變的內容
雜湊快取。

`zyenv` 繼續只負責選擇編譯器版本，不安裝專案函式庫。

## 命令

```text
zy pkg init
zy pkg add httpx
zy pkg add github:Ryan-2013/zy-raylib
zy pkg add ../local-library
zy pkg remove httpx
zy pkg install
zy pkg update [package]
zy pkg list
zy pkg publish
```

`zy pkg add` 同時更新 manifest 與 lockfile；`zy pkg install` 只安裝鎖定的
依賴圖。`--locked` 拒絕 manifest 漂移，`--offline` 拒絕任何需要網路的操作。

## Manifest

專案 manifest 名為 `zyproject.toml`：

```toml
[package]
name = "my-game"
version = "0.1.0"
entry = "src/main.zy"
zyen = ">=0.1.84,<0.2"

[dependencies]
httpx = "^1.2.0"
raylib = { git = "https://github.com/Ryan-2013/zy-raylib", rev = "..." }
local-ui = { path = "../local-ui" }
```

Registry 版本採 semantic versioning；Git 依賴鎖定 commit；path 依賴只供開發，
發布前必須替換。

## Lockfile 與解析

`zy.lock` 記錄精確版本或 commit、來源 URL、依賴邊與 SHA-256 archive digest。
整個專案對每個 package name 只選一個版本。第一版 resolver 使用確定性的
回溯，優先選擇最高相容版本；衝突診斷顯示造成不相容限制的最短依賴鏈。

只有 `add`、`remove`、`update` 會主動改 lockfile。check、run、build 不會
偷偷選擇新版本。

## 儲存與 import

Archive 依內容雜湊保存，且不可變：

```text
~/.zyen/packages/v1/<sha256>/
```

編譯器讀取 `zy.lock`，直接從快取解析套件 import：

```zy
import <httpx/client> as http;
```

相對 import 不得逃出 package root；專案不需要把依賴原始碼複製進 repo。

## Native 套件與安全

Native 套件和 `std/c_module` 使用完全相同的公開 `.zlcm.h`、`.h`、`.c`
機制。安裝過程不執行套件程式碼、compiler plugin 或 build script。平台設定
使用既有的 `ZLC_SOURCE_*`、`ZLC_HEADER_*`、`ZLC_LIB_*`、`ZLC_CFLAG_*`。

Registry 下載使用 HTTPS，並以 lockfile 的 SHA-256 驗證。發布後的
name/version 組合不可變。第一版不支援預編譯 native binary 或安裝腳本。

## 交付階段

1. 本機 manifest、path dependency、lockfile、cache、compiler resolver。
2. 鎖定 commit 的 Git dependency，以及 offline、locked mode。
3. 唯讀 registry、add/install/update 與衝突診斷。
4. 登入、所有權、publish、yank、search。

每一階段都要測試 Windows、Linux、macOS，而且 portable compiler 不得要求
系統已安裝 Python。
