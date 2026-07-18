# ZyenLang Enhancement Proposals (ZEPs)

[English](README.md) | **繁體中文**

> **相關 repo**
> - **編譯器 & stdlib**:[zyenlang-v0.1.49](https://github.com/Ryan-2013/zyenlang-v0.1.49)
> - **IDE(dogfood)**:[zyenlang-ide](https://github.com/Ryan-2013/zyenlang-ide)

一套小而明確的 ZyenLang 規格系統,參考 Python PEP 的設計,但只留下單作者
+ AI 協作真正用得到的部分。

每份 ZEP 是一個獨立的 markdown 檔。共三種類型:

| 類型 | 用途 | 範例 |
|---|---|---|
| **Process(流程)** | 專案怎麼跑 | ZEP-0001(目的與流程) |
| **Informational(資訊)** | 慣例、最佳實務、風格指南 | ZEP-0003(風格指南) |
| **Standards(標準)** | 編譯器或 stdlib 必須強制執行的規矩 | ZEP-0004(全域狀態) |

把已經 ship 的行為固化下來的 ZEP 標為 `Active`(Process / Informational)
或 `Final`(Standards)。提案中的 ZEP 在被接受前是 `Draft`。

## 索引

| ZEP | 標題 | 狀態 | 類型 |
|---|---|---|---|
| [0001](ZEP-0001-purpose-and-process.zh-TW.md) | 目的與流程 | Active | Process |
| [0002](ZEP-0002-zep-template.zh-TW.md) | ZEP 範本 | Active | Process |
| [0003](ZEP-0003-style-guide.zh-TW.md) | ZyenLang 風格指南 | Active | Informational |
| [0004](ZEP-0004-global-state-and-constants.zh-TW.md) | 全域狀態與模組層級常數 | Final | Standards |
| [0005](ZEP-0005-error-handling.zh-TW.md) | 錯誤處理慣例 | Active | Informational |
| [0006](ZEP-0006-struct-field-defaults.zh-TW.md) | 結構欄位預設值 | Final | Standards |
| [0007](ZEP-0007-zy-doctor.zh-TW.md) | `zy doctor`:系統健檢 | Active | Standards |
| [0008](ZEP-0008-zyenv.zh-TW.md) | `zyenv`:版本管理器 | Active | Standards |
| [0010](ZEP-0010-first-class-functions.zh-TW.md) | 一等公民函式值 | Active | Standards |
| [0011](ZEP-0011-struct-body-methods.zh-TW.md) | 結構體內方法定義 | Final | Standards |
| [0013](ZEP-0013-closures.zh-TW.md) | 閉包與 Lambda Lifting | Active | Standards |
| [0014](ZEP-0014-managed-pointers-and-arc.zh-TW.md) | 受管指標與自動參考計數 | Final | Standards |

## 怎麼讀

剛接觸 ZyenLang 的人請照順序讀:0001 講系統本身,0002 給你看 ZEP 長怎樣,
0003 是日常風格,0004 解釋為什麼不能有頂層變數,0005 是錯誤怎麼穿過 stdlib。

要寫新 ZEP,複製 `ZEP-0002-zep-template.md`,編號往下推,提交。

## 不是 ZEP 的東西

- Bug 回報 —— 放在編譯器 repo 的 `docs/`。
- 個別模組 API 文件 —— 放在 `docs/std_<module>.md`。
- 變更日誌與版本筆記 —— 放在 `SPEC_v0_1.md` 和 release notes。

ZEP 紀錄的是**設計原則**和**長期慣例**。如果一條規則在 README 用一行就講完了,
就不需要寫 ZEP。
