# ZEP-0007 — `zy doctor`:系統健檢

[English](ZEP-0007-zy-doctor.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0007 |
| **Title** | `zy doctor`: System Health Check |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Standards |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

一個唯讀的健檢指令,診斷 ZyenLang 使用者最常遇到的安裝、環境、設定問題。
參考 `flutter doctor` 與 `brew doctor`。用 Python 實作,作為 `zy` CLI 的
子指令發布。**永遠安全,絕不修改使用者狀態**。

## 動機

當 `zy build` 或 `zy run` 失敗,原因幾乎都是下面其中之一:

- `gcc` 不存在或不在 `PATH` 上
- Python 版本低於最低需求
- `tkinter` 缺失(只有 GUI IDE 需要)
- 兩個 `std/` 鏡像不同步
- 已安裝的 `zyenlang` package 比 source tree 還舊
- 工作目錄的路徑 / 權限問題

每一條的診斷都很明顯,但使用者要**知道**該檢查什麼。`zy doctor` 把診斷
壓縮成一個指令,任何使用者出問題時的第一步都可以跑。

它也是未來 ZyenLang 生態系工具(`zyenv` 版本管理、`zypm` 套件管理器)的
**bootstrap 層**:兩者都會假設 `zy doctor` 通過後才嘗試網路操作。

## 規範

### 指令

```
zy doctor [-v | --verbose] [--json] [--no-runtime]
```

### 分類與檢查項

檢查依優先順序分四個類別(環境第一,因為其他東西都依賴它;runtime 最後,
因為它最貴)。

#### 環境(Environment)

| ID | 檢查內容 | 失敗時嚴重度 |
|---|---|---|
| `python_version` | Python 直譯器 ≥ 3.8 | **error** |
| `tkinter` | `import tkinter` 成功 | **warning**(CLI 不用它也能跑) |
| `cc` | `PATH` 上有 C 編譯器(優先 gcc,接受 clang) | **error** |
| `cc_compiles` | C 編譯器能從 `int main(){return 0;}` 產生可執行檔 | **error** |
| `zy_on_path` | `zy` 指令在 `PATH` 上 | **warning**(用 `python -m zyenlang` 也行) |

#### 安裝(Installation)

| ID | 檢查內容 | 失敗時嚴重度 |
|---|---|---|
| `package_installed` | `import zyenlang` 成功 | **error** |
| `package_version` | 回報已安裝版本 | informational |
| `transpiler_importable` | `from zyenlang import transpiler` 成功 | **error** |
| `std_modules_count` | `zyenlang/std/` 至少有 40 個 `.zy` 模組 | **error** |
| `std_mirror_sync` | 外層 `std/` 與內層 `zyenlang/std/` byte-identical | **warning** |

#### 設定(Configuration)

| ID | 檢查內容 | 失敗時嚴重度 |
|---|---|---|
| `vscode_ext` | VS Code 擴充已安裝 | informational |
| `zed_support` | Zed 編輯器支援已安裝 | informational |
| `tmpdir_writable` | session 暫存目錄(例如 `ide_session/`)可寫 | **warning** |

#### Runtime(用 `--no-runtime` 跳過)

| ID | 檢查內容 | 失敗時嚴重度 |
|---|---|---|
| `smoke_transpile` | 一支極小的 `.zy` 程式能轉譯為 `.c` | **error** |
| `smoke_compile` | 產生的 `.c` 能編譯出執行檔 | **error** |
| `smoke_run` | 執行檔能跑且輸出預期內容 | **error** |

### 嚴重度

- **informational** —— 永遠不影響 run,只是用來回報版本之類資訊。
- **warning** —— 不阻止工具鏈使用,但該修。
- **error** —— 至少一個核心功能壞掉。

### 人類輸出格式

```
ZyenLang doctor — v0.1.49

Environment
  ✓  python 3.11.5
  ✓  tkinter
  ✓  gcc 13.2.0
  ✓  zy on PATH

Installation
  ✓  zyenlang 0.1.49 installed
  ✓  transpiler importable
  ✓  43 stdlib modules
  !  std/ and zyenlang/std/ differ in 1 file:
       std/mem.zy missing alloc_any()
     Fix: cp zyenlang/std/mem.zy std/mem.zy

Configuration
  ✓  VS Code extension installed
  -  Zed support not installed (optional)
     Hint: python tools/install_zed_support.py

Runtime
  ✓  smoke transpile
  ✓  smoke compile
  ✓  smoke run

Summary: 11 passed, 1 warning, 0 errors
```

符號:`✓` pass、`!` warning、`✗` error、`-` informational。

### JSON 輸出格式

`--json` 每個檢查發出一筆機器可讀物件,加上最後一筆 summary,適合 CI 把關
與 IDE 狀態面板:

```json
{
  "doctor_version": 1,
  "zyenlang_version": "0.1.49",
  "checks": [
    {
      "id": "python_version",
      "category": "environment",
      "status": "pass",
      "detail": "3.11.5"
    },
    {
      "id": "std_mirror_sync",
      "category": "installation",
      "status": "warning",
      "detail": "std/mem.zy missing alloc_any()",
      "fix": "cp zyenlang/std/mem.zy std/mem.zy"
    }
  ],
  "summary": { "pass": 11, "warning": 1, "error": 0 }
}
```

### Exit code

| Code | 意義 |
|---|---|
| **0** | 全部通過(informational 不影響 exit code)。 |
| **1** | 至少一個 warning,沒有 error。 |
| **2** | 至少一個 error。 |

這跟 ZEP-0005(錯誤是值)一致:CI 裡呼叫 `zy doctor` 的工具可以單看 exit code
分支,不必 parse 輸出。

### Flags

| Flag | 效果 |
|---|---|
| `-v`、`--verbose` | 每個檢查多印路徑、版本、額外 context。 |
| `--json` | 改 emit JSON。warning/error 仍對應上面的 exit code。 |
| `--no-runtime` | 跳過 smoke trio(省 ~3 秒;CI doctor pass 適用)。 |

### `zy doctor` **不做**的事

- **v1 沒有 `--fix` flag**。Doctor 報告,使用者自己修。理由見下面 *設計理由*。
- **沒有網路檢查**。GitHub / 未來 `zypm` registry 的可達性是那些工具自己的事。
- **沒有專案層級檢查**。Doctor 管的是工具鏈,不是使用者 `.zy` 原始碼
  (那是 `zy check <file>` 的職責)。
- **沒有 plugin / 自訂檢查**。檢查項是固定且權威的;沒被覆蓋的失敗應該作為
  本 ZEP 的 bug 回報,而不是在使用者端 patch。

## 設計理由

### 為什麼用 Python 不用 ZyenLang

`zy doctor` 是 ZyenLang 壞掉時使用者跑的第一個指令。如果 doctor 本身就是
ZyenLang 程式,ZyenLang 壞掉時 doctor 也跑不起來。Python 是 bootstrap 層 ——
跟 Rust 的 `rustup` 早期不用 Rust 寫、`pyenv` 不用 Python 寫是同一個道理。

這個決定**不可**被未來 ZEP 反轉,除非每個平台都有 native binary 預編 doctor
的可行路徑。

### 為什麼唯讀(v1 沒 `--fix`)

`--fix` flag 會讓 doctor 修改使用者狀態(覆寫檔案、安裝套件)。這把影響範圍
從「報告」升級為「需要確認的破壞性動作」。我們刻意把 v1 維持唯讀,並為每個
warning / error 列出修法讓使用者複製貼上。`--fix` flag 可能在後續 ZEP 加入,
等報告面成熟再說。

### 為什麼有 smoke-run 檢查

靜態檢查(檔案存在嗎?import 成功嗎?版本有達標嗎?)會漏掉細微環境問題:
錯的 libc 版本、gcc codegen 壞掉、Windows 防毒軟體擋執行檔啟動。Smoke trio
(transpile → compile → run)是唯一**端到端**走完整 pipeline 的檢查。它抓得
到靜態檢查抓不到的東西。

Runtime 檢查也是最常在 CI 為了效能跳過的那個,所以才有 `--no-runtime` flag。

### 為什麼把檢查分類

14 個檢查平鋪不好掃。按**驗證的 stack 層級**分組讓使用者依優先順序掃過 ——
環境第一、runtime 最後。心智模型對應依賴圖 —— 修好環境錯誤常常會清掉下面的
安裝錯誤,所以應該先報告。

### 為什麼最低 Python 3.8

walrus 運算子(3.8)在 transpiler 裡已用。降到 3.8 以下要重寫熱路徑。
Python 3.7 在 2023 就 upstream EOL;3.8 是新開發的實務地板。Doctor 把
低於 3.8 視為 hard error,免得收到不相容環境的 bug 回報。

### 為什麼 `tkinter` 是 warning 不是 error

CLI 編輯器(`ide.zy`)不需要 `tkinter`。GUI 編輯器(`ide_gui.zy`)需要。
我們無從知道使用者要用哪個,所以給 warning 不給 error。Warning 文字會點名
是哪個子指令需要這個缺失的依賴。

## 向後相容性

完全向後相容。`zy doctor` 是新子指令。沒有既有行為被改。

Exit code 政策(0 / 1 / 2)是新的,但只影響明確呼叫 `zy doctor` 的腳本。
今天沒有這種腳本存在。

## 參考實作

於 2026-06-20 在 `zyenlang-v0.1.49` shipped:

- `zyenlang/cli/__init__.py` —— package marker。
- `zyenlang/cli/doctor_checks.py` —— 每個 check ID 一個函式,各自回傳
  `CheckResult` dataclass(id、category、status、detail、fix)。共 16 個
  檢查(5 environment、5 installation、3 configuration、3 runtime)。
- `zyenlang/cli/doctor.py` —— 入口、人類版與 JSON 格式化器、exit code
  對映。也可以直接 `python -m zyenlang.cli.doctor`。
- `zyenlang/transpiler.py` —— `main()` 透過
  `cli.doctor.add_subparser(sub)` 接 `doctor` 子指令,並在既有 input-file
  路徑前用 `cli.doctor.handle(args)` dispatch。
- `tests/test_doctor_python.py` —— 五個 smoke test,驗證 JSON envelope
  形狀、必要 check ID 完整集合(有與沒有 `--no-runtime` 兩種)、每筆檢查
  結構完整,以及 exit code 與 summary 一致。

開放後續工作:測試套件目前是「驗證輸出形狀」,不是「捏造壞環境並斷言對應檢查
失敗」。模擬 subprocess 與 importlib 的 fixture 套件能加強回歸覆蓋。如需要,
可開後續 ZEP 追蹤。

## 待釐清問題

- 是否在後續 ZEP 加 `zy doctor --fix` flag。目前刻意排除以維持 v1 表面小。
- 是否該支援使用者自訂的 JSON config,可以關掉特定檢查(如 CI 裡
  `--skip vscode_ext`)。可能 v1.1 加;本 ZEP 不含。
- `zy_on_path` 檢查是否該偵測版本不一致(使用者 `PATH` 上有另一個 install
  的 `zy`)。
- 人類輸出是否該有顏色。預設:stdout 是 TTY 時給,pipe 或 `--json` 時不給。

## 參考

- `flutter doctor` —— 功能面最接近的模型。
- `brew doctor` —— 次要靈感來源,特別是 warning vs error 的區分。
- [ZEP-0005](ZEP-0005-error-handling.zh-TW.md) —— doctor exit code 遵循
  的 int-code 慣例。
- [ZEP-0001](ZEP-0001-purpose-and-process.zh-TW.md) —— Status 定義。

## 版權

本文件置於公有領域。
