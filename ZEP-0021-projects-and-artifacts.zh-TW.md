# ZEP-0021：專案、依賴與產物 target

[English](ZEP-0021-projects-and-artifacts.md) | **繁體中文**

| 欄位 | 內容 |
|---|---|
| **ZEP** | 0021 |
| **標題** | 專案、依賴與產物 target |
| **作者** | zuenchen、OpenAI Codex |
| **狀態** | Final |
| **類型** | Standards |
| **建立日期** | 2026-09-18 |
| **取代** | [ZEP-0017](ZEP-0017-package-manager.zh-TW.md) |
| **依賴** | [ZEP-0020](ZEP-0020-v0.3-core-language.zh-TW.md) |
| **參考實作** | `Ryan-2013/zyenlang` 的 `v0.3.0` tag |

## 摘要

ZyenLang 0.3 把專案建立、依賴鎖定、target 選擇、輸出位置與安全清理整合進
扁平的 `zy` 命令。舊 `zy pkg` 命令族已移除。resolver 支援 registry entry、
local path 與鎖定精確 revision 的 Git repository，且不執行 dependency build
或 install script。

## 命令

```text
zy new PATH [--lib]
zy init [PATH] [--lib]
zy install [NAME[==VERSION] | PATH | git+URL@COMMIT] [--alias ALIAS]
zy uninstall ALIAS
zy list
zy show NAME
zy add ALIAS --path PATH
zy add ALIAS --git URL --rev COMMIT
zy remove ALIAS
zy fetch [--locked]
zy check [TARGET]
zy build [TARGET] [--release] [--out-dir PATH]
zy run [TARGET] -- [ARGS...]
zy test
zy clean [TARGET]
zy metadata
zy emit --file SOURCE --kind c --out-dir PATH [--output-name NAME]
```

編輯器與單檔檢查可用 `zy check --file SOURCE [--library]`。舊單檔
`zy build source.zy -o output` 已移除，因為副檔名不可決定 artifact kind；明確的
非專案 C 輸出使用 `zy emit`。

`zy install` 不帶 requirement 時解析目前 manifest；帶 requirement 時新增並
鎖定 dependency。`zy add`、`zy remove`、`zy fetch` 保留為低階相容命令。

`zy run` 執行 project binary 時以 executable output directory 作為 working
directory。標準 filesystem operation 即使從其他 working directory 啟動已建置的
binary，也一律從 executable directory 解析相對 path。

## Manifest 與 target

Manifest 名為 `zyproject.toml`：

```toml
[package]
name = "zy-math"
version = "0.3.0"
zyen = ">=0.3.0"
entry = "src/lib.zy"

[build]
default-target = "ffi"
target-dir = "target"

[targets.app]
kind = "bin"
entry = "src/main.zy"

[targets.ffi]
kind = "c-source"
entry = "src/lib.zy"
out-dir = "../consumer/generated"
output-name = "zy_math"

[dependencies]
utils = { path = "../utils" }
net = { git = "https://example.com/net.git", rev = "COMMIT_SHA" }
```

支援四種 kind：

- `bin`：需要 `fn main() i32` 的原生執行檔；
- `c-source`：C source/header/runtime/native bundle 與 metadata；
- `staticlib`：平台 static library 與 public header；
- `sharedlib`：平台 shared library、public header，以及平台需要的 import library。

預設輸出至 `target/debug/<target>/` 或 `target/release/<target>/`。target 的
`out-dir` 相對 project root；CLI `--out-dir` 優先級最高。`output-name` 預設為
target key。artifact kind 不由檔名副檔名推測。

只有 `export fn` 進入生成的 C/C++ header。0.3 的直接 export 僅支援不含泛型、
不 throws，且使用固定寬度數字、`bool`、`void`、`ZL_String` 的函式；複雜 API
需提供受支援的 facade。

## 依賴圖與 import

dependency key 就是 import alias。path dependency 相對擁有它的 manifest 解析；
Git dependency 必須指定精確 revision，lock 會保存 resolved commit。只 import
dependency alias 時載入其 manifest entry；額外 `::` segment 解析 source
submodule。`as` 可省略，預設使用 path 最後一段。package 只可 import 自己的
direct dependency：

```zy
import utils
import utils::math as math
let version = utils::version()
let answer = math::add(20, 22)
```

import 必須留在 dependency 的 `src/` root 內。resolver 拒絕路徑穿越、symlink、
dependency cycle、alias collision、不安全名稱、可變 Git revision 語法、不支援
的 URL，以及超過 graph/file/byte 限制的 package。

`zy.lock` 記錄 root manifest digest，以及每個 package 的 alias、身份、source、
精確 commit、content digest 與 transitive edges。解析順序必須確定。
`zy fetch` 更新 lock；`zy fetch --locked`、check、build、run 遇到缺少、過期或
內容不一致的 lock 必須失敗，不可偷偷改寫。

Package snapshot 以內容雜湊保存在 `~/.zyen/packages/0.3/<sha256>/`；Git source
移除 repository metadata 後保存在 `~/.zyen/git/0.3/`。

Registry schema 1 是有大小限制的 JSON index。package entry 包含 `latest`
version，以及 exact version 到已驗證 Git URL 與完整 commit 的 mapping。
`zy install name` 選擇 `latest`，`zy install name==version` 選擇指定版本；取得的
manifest name/version 必須和 index 相符。預設 index 使用 HTTPS；私人或離線環境
可用 `ZYEN_REGISTRY` 指定另一個 HTTPS URL 或 local index。Registry data 不可直接
提供 shell command 或 compiler flag。

## Test、metadata 與安全 clean

`zy test` 優先執行名稱為 `test` 或以 `test-` 開頭的 target；若沒有，就依序
執行 `tests/**/*.zy`，遇到非零結果即停止。`zy metadata` 不編譯程式，輸出可供
工具讀取的 package、target、build 與 locked dependency 資料。

每次成功 build 都記錄它建立的精確 absolute file。`zy clean [target]` 只刪除
這些檔案，拒絕 relative path、malformed record、symlink 與 directory，絕不遞迴
刪除 `out-dir`，即使它是含其他 consumer 檔案的外部目錄。

## 安全與範圍

fetch dependency 不會執行 source 或 install hook。精確 revision 與 SHA-256
digest 提供可重現性及漂移偵測，不代表 native dependency 安全；build/run 會以
目前 process 權限信任其 C input。

0.3 不提供 semver range solver、自動 publish service、build script、dependency
hook、prebuilt native package format 或 re-export。
