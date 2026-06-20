# ZEP-0001 — 目的與流程

[English](ZEP-0001-purpose-and-process.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0001 |
| **Title** | Purpose and Process |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Process |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

本 ZEP 定義什麼是 ZEP、什麼**不是** ZEP、提案的生命週期,以及作者和審閱者
的責任。涉及語法或 stdlib 行為的 ZEP 具有約束力:編譯器與 stdlib 必須遵守。

## 動機

ZyenLang 是一個小型實驗語言,一個人類作者加一群輪換的 AI 助手。語法與標準
函式庫的決定散落在 commit message、README、CLAUDE.md、聊天紀錄裡。剛開始
幾個月還行,之後爛得很快:下一個貢獻者(或半年後的同一個作者)看不出哪些
決定是承重的、哪些只是當下的權宜之計。

ZEP 給每個長期決定一份穩定、有日期、有編號的文件,以此解決上述問題。新貢獻者
讀 ZEP 索引,不必去翻 git log。未來的你可以反對某個 ZEP,但至少反對是明確的,
不是不小心反掉。

## 規範

### 類型

| 類型 | 用途 |
|---|---|
| **Process(流程)** | 治理、生命週期、誰決定什麼。 |
| **Informational(資訊)** | 慣例、風格指南、推薦寫法。編譯器不強制。 |
| **Standards(標準)** | 編譯器或 stdlib 必須強制的規則。違反 = bug。 |

`Standards` 是最強的形式。一旦進入 `Final`,編譯器就被它綁住。`Informational`
是強烈建議;編譯器不會拒絕違反者,但 stdlib 內部必須遵守。

### 狀態

```
Draft  →  Review  →  Active   (Process / Informational,持續生效)
                  →  Final    (Standards,鎖定)
                  →  Rejected (未合併直接關掉)
                  →  Withdrawn (作者收回)

以上任一(除 Draft 外) →  Superseded  (被更新的 ZEP 取代)
                       →  Deprecated  (仍有效但已不推薦)
```

- **Draft**:還在寫。沒有約束力。
- **Review**:作者覺得寫好了,正在收集反對意見。
- **Active**:生效中,內容如無實質變動可原地修訂。
- **Final**:生效中且鎖定。實質修改需要新開一份 ZEP Supersede 它。
- **Rejected** / **Withdrawn**:留在 repo 裡作為紀錄。不適用。

### 編號

ZEP 從 1 開始連續編號,檔名中補零至四位:
`ZEP-0001-purpose-and-process.md`。

編號一旦發出就不重用、不重指派。被撤回或駁回的 ZEP 保留其編號。

### 作者

任何人都可以提交 ZEP。作者可以是 `zuenchen`(人類維護者)、具名的 AI 助手
(`Claude Opus 4.7`、`ChatGPT`)、或未來的外部貢獻者。多位作者用逗號分隔。

維護者(`zuenchen`)是狀態轉換的最終決定者。

### 工作流程

1. **Draft**:複製 `ZEP-0002-zep-template.md`,取下一個空編號,寫提案。
   Status:`Draft`。
2. **Review**:寫好之後改 Status 為 `Review`,開 PR 或公開徵求反饋。
3. **Decision**:維護者標為 `Active` / `Final` / `Rejected` / `Withdrawn`。
4. **Index**:把 ZEP 加進 `README.md` 的索引表。

### 不是 ZEP 的東西

下列東西不需要寫成 ZEP:

- Bug 回報(放在編譯器 repo 的 `docs/`)。
- 個別模組 API 參考(放在 `docs/std_<module>.md`)。
- Release notes 或 changelog(放在 `SPEC_v0_1.md`)。
- 一句話就能講完的規則(放在 README)。
- 短的風格偏好(擴充既有的風格指南 ZEP)。

不確定就直接寫成 ZEP。多一份 ZEP 的成本很低;沒文件的決定成本很高。

## 為什麼是縮水版治理

PEP 風格的治理是為了多作者、上千貢獻者的語言設計的。ZyenLang 只有一個人類
加幾個 AI 助手。本 ZEP 拿掉了投票統計、BDFL 代理、PEP-Editor 角色,只剩下:

- 一個固定編號(這樣我們可以永遠引用「ZEP-0004」)。
- 一個狀態(這樣讀者知道文件是否仍適用)。
- 一份範本(這樣 ZEP 都好翻)。

當專案長大,這份 ZEP 自己可以被一份更完整的治理文件 Supersede 掉。

## 參考

- [PEP 1 — PEP Purpose and Guidelines](https://peps.python.org/pep-0001/)
  (本 ZEP 縮水自此)

## 版權

本文件置於公有領域。
