# ZEP-0008 — `zyenv`:ZyenLang 版本管理器

[English](ZEP-0008-zyenv.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0008 |
| **標題** | `zyenv`:ZyenLang 版本管理器 |
| **作者** | zuenchen、Claude Opus 4.7 |
| **狀態** | Active |
| **類型** | Standards |
| **建立日期** | 2026-06-21 |
| **修訂歷史** | 2026-06-21 |

## 摘要

一個 CLI 工具,讓單一使用者可以同時裝多個 ZyenLang 版本,並以
shell / 目錄 / 全機器為單位切換當前版本。設計參考
`pyenv` / `rbenv` / `nvm`。Python 實作,以 `zyenlang` 套件的
`zyenv` console script 方式發佈。

## 動機

「把專案釘住某個 ZyenLang 版本」是反覆會出現的需求:

- 重現舊版回報的 bug,但不能丟掉現有的安裝。
- 並排比較不同版本的語言行為(例如 `v0.1.43` 對 `v0.1.49`)。
- 主專案綁定 v0.1.49,但實驗性的旁支需要一個還沒 release 的版本。

`zyenv` 出現之前,只能 `pip install` 一個版本、用完再 uninstall、
裝下一個。「切換」既慢、又會丟東西、又是各種「在我這邊可以跑」
的隱形根因。

`zyenv` 把流程壓成:

```
zyenv install-local D:\path\to\zyenlang-v0.1.43
zyenv install-local D:\path\to\zyenlang-v0.1.49
zyenv use v0.1.49        # 全機器預設
cd my_legacy_project
zyenv local v0.1.43      # 這個目錄釘 v0.1.43
```

PATH 上的 shim 會把 `zy`、`zyen` 的呼叫路由到 `zyenv` 為當前
shell + cwd 解析出來的那一版。

## 規範

### 目錄配置

```text
$ZYENV_HOME             預設:~/.zyenv
├── versions/
│   ├── v0.1.43/
│   │   └── venv/       完整 Python venv,裡面裝了那一版的 zyenlang
│   ├── v0.1.49/
│   │   └── venv/
│   └── ...
├── shims/
│   ├── zy.bat          Windows shim(Unix 是 `zy`,沒副檔名)
│   ├── zyen.bat
│   └── zy_resolver.py  純 stdlib Python 解析器(不需要裝 zyenlang)
└── version             可選的單行檔:全機器預設版本
```

任何專案目錄(或 cwd 的任何祖先)裡的 `.zy-version` 會覆蓋全機器
預設。`ZY_VERSION` 環境變數可以在單一 shell 內覆蓋一切。

### 子命令

| 命令 | 行為 |
|---|---|
| `zyenv init` | 建 `~/.zyenv/{versions,shims}`、寫 shim、印出該 shell 該怎麼改 PATH。 |
| `zyenv list` / `versions` | 列出已安裝版本,當前那個用 `*` 標記。 |
| `zyenv current` / `version` | 列當前版本以及它是怎麼被選到的(`shell` / `local` / `global`)。 |
| `zyenv use <version>` | 寫全機器預設到 `~/.zyenv/version`。 |
| `zyenv local [<version>]` | 有參數:在 cwd 寫 `.zy-version`。沒參數:印當前 local pin。 |
| `zyenv unset` | 從 cwd 刪掉 `.zy-version`。 |
| `zyenv which` / `path` | 印 shim 會呼叫的 `zy` 真實絕對路徑。 |
| `zyenv install-local <path> [--as NAME] [--force]` | 在 `versions/<NAME>/venv` 建新 venv,把 `<path>` 用 `pip install -e` 灌進去。`--as` 預設讀 `pyproject.toml` 的 `vX.Y.Z`。 |
| `zyenv install-zip <path> [--as NAME] [--force]` | 解壓含 Python package 的 `.zip` 後同樣安裝。 |
| `zyenv uninstall <version>` | 刪掉該版本目錄;如果全域指標指向它,連帶清除。 |

### 版本解析順序

優先序由高到低:

1. `ZY_VERSION` 環境變數。
2. 從 cwd 往根目錄走,第一個 `.zy-version` 檔。
3. `$ZYENV_HOME/version`(全機器預設)。

如果三者都沒解析到,shim 用 status 1 退出並印一條清楚訊息。

### Shim 機制

Shim 是純 stdlib 的 Python script
(`~/.zyenv/shims/zy_resolver.py`),做的事:

1. 依上面的解析順序找到 version。
2. 定位 `versions/<resolved>/venv/{Scripts,bin}/{zy.exe,zy}`。
3. `subprocess.run` 它,把 `sys.argv[1:]` 帶過去,回傳 exit code。

Windows 上有個小小的 `zy.bat` 包裝 `python "%~dp0\zy_resolver.py" %*`,
所以 PATH 上看到的指令名稱仍是 `zy`。Unix 上則直接是 `zy` 檔,
頂上是 shebang。`zyen.bat` / `zyen` 為了 legacy 別名也各複製一份。

Shim 依賴系統 PATH 上有個 Python(3.10+)。它**不**依賴任何
ZyenLang 安裝,所以某個版本壞掉不會搞死 shim。

### 安裝隔離

每個安裝的版本住在自己的 venv 裡。沒有共用的 site-packages。
裝一個版本帶進來的副作用(transitive deps、runtime header 生成)
不會跨版本汙染。

### 網路

`zyenv install-local` 跟 `install-zip` 用
`pip install --no-build-isolation`。新 venv 預裝的 `setuptools`
能滿足 build-system 需求,不必連 pypi.org —— 所以在 SSL 受限的
網路(企業內網會劫持憑證)也能裝。如果目標 package 宣告了 runtime
deps 且 venv 沒有,pip 仍然會嘗試下載;這種情況使用者得自己預先
`pip install --target` 或接受安裝失敗。

## 設計理由

### 為什麼用 per-version venv 而不是直接 site-packages

三個原因:

1. **清理就是 rm -rf** 一個目錄。沒有「pip 殘留」。
2. **沒有 site-packages 串味。** 兩個版本可以宣告會衝突的 C runtime
   module 卻不互相打架。
3. **Shim 只有一個工作:**指向固定的 binary 路徑。沒有 PYTHONPATH
   體操、沒有 module-resolution 驚喜。

### 為什麼 shim 是 Python 不是原生

原生 shim(C 執行檔或 shell script)每個 OS 都得自己實作檔案系統
往上爬 + UTF-8 處理。純 stdlib 的 Python script 80 行內、Windows /
macOS / Linux 都一模一樣跑得起來。Python 約 50 ms 的啟動時間,對
一個本身就要呼叫 `gcc` 的 CLI 工具感覺不出來。

### 為什麼預設 `--no-build-isolation`

PEP 517 build isolation 會建一個臨時 venv,然後從 pypi 灌
`[build-system].requires`。受限網路會在這一步就失敗,根本還沒摸到
使用者的 code。ZyenLang 唯一的 build dep 是 `setuptools >= 68`,
這恰好是 `venv.create` 預裝的。跳過 isolation 在常規情況下換來
「完全離線也能裝」的能力。

### 為什麼版本解析是單一鏈、沒有 fallback

`pyenv` 提供一條 chain(`pyenv versions` 列當前排序好的列表加當前
那個)。`zyenv` 每次呼叫回傳唯一一個解析到的版本。允許 fallback
(「v0.1.49 不在,fallback 到 v0.1.43」)等於邀請隱形的版本漂移,
那正是版本管理器存在的反面。被釘的版本沒裝,shim 就拒跑、告訴
使用者怎麼裝。

## 向後相容性

完全 additive。從不呼叫 `zyenv` 的使用者完全感覺不到改變。既有的
`pip install -e .` 系統安裝照常運作;一旦把 `~/.zyenv/shims` 加到
PATH、排在系統 Python 的 `Scripts` 目錄前面,`zy` 呼叫就改走 shim。

如果系統有 `zyenlang` 安裝**且** shim 沒選版本,使用者會看到 shim
的「沒選版本」錯誤,而不是系統 `zy`。建議:

- 用 `zyenv use vX.Y.Z` 把 shim 指向受管理的版本,或
- 想用系統安裝的 shell 把 `~/.zyenv/shims` 從 PATH 拿掉。

## 參考實作

實作在 `zyenlang-v0.1.49` 的 `zyenlang/cli/zyenv.py`。純 stdlib
的 resolver 模板是該模組底部的 `_SHIM_PY` 字串;`zyenv init` 會把
它寫到 `~/.zyenv/shims/zy_resolver.py`,確保 shim 開機從不需要
ZyenLang 或任何第三方 Python 套件。

`zyenv` console_script 註冊在 `pyproject.toml` 的 `[project.scripts]`:

```toml
zyenv = "zyenlang.cli.zyenv:main"
```

`zyenlang` 和 `zyenlang.cli` 都列在 `[tool.setuptools].packages`,
讓 cli 子模組進 wheel。

## 待解問題

- **遠端安裝。** `install-remote vX.Y.Z`(從 GitHub release 或
  pypi)刻意延後 —— 使用者現在主要在 iterate 本地 source tree,
  workflow 不需要。
- **`zyenv exec <cmd>`。** 仿 `pyenv exec`,用解析到的版本環境跑
  一個命令。對 one-off script 有用。
- **`zyenv rehash`。** Shim 是「一個 binary 名稱一個檔案」(`zy`、
  `zyen`)。日後若加更多 entry point,這條命令重掃 + 寫新 shim 檔。
- **裝了什麼版本的 telemetry。** `zyenv doctor` 可以跟
  [ZEP-0007](ZEP-0007-zy-doctor.zh-TW.md) 串起來 —— 留給後續 ZEP。

## 參考資料

- [ZEP-0007](ZEP-0007-zy-doctor.zh-TW.md) —— `zy doctor` 開了
  「CLI 工具住在 `zyenlang/cli/`」的先例。
- pyenv (https://github.com/pyenv/pyenv)、rbenv
  (https://github.com/rbenv/rbenv)、nvm
  (https://github.com/nvm-sh/nvm) —— 直接的先驅。
- PEP 517 / PEP 518 —— `--no-build-isolation` 退出的就是這個
  build-isolation 模型。

## 版權

本文件置於公有領域。
