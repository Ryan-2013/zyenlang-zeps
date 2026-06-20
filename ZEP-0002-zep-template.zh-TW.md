# ZEP-0002 — ZEP 範本

[English](ZEP-0002-zep-template.md) | **繁體中文**

| 欄位 | 值 |
|---|---|
| **ZEP** | 0002 |
| **Title** | ZEP Template |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Process |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## 摘要

寫新 ZEP 時複製的 boilerplate。下面的章節順序請保留;不適用的章節可刪,
但不要重排。

## 怎麼用這份範本

1. 複製本檔。
2. 改名為 `ZEP-NNNN-short-kebab-title.md`。`NNNN` 用下一個空整數,
   補零至四位。
3. 取代 header 欄位與各章節內容。
4. 在 `README.md` 的索引表加一筆。

---

# ZEP-NNNN — 簡短標題

| 欄位 | 值 |
|---|---|
| **ZEP** | NNNN |
| **Title** | Short Title |
| **Author** | 你的名字 |
| **Status** | Draft |
| **Type** | Standards / Informational / Process |
| **Created** | YYYY-MM-DD |
| **Post-History** | YYYY-MM-DD |
| **Supersedes** | (選填)ZEP-MMMM |
| **Superseded-By** | (選填,稍後設)ZEP-OOOO |

## 摘要

一段話講完。本 ZEP 決定了什麼。讀者掃過摘要就應該知道剩下的內容是否與他有關。

## 動機

為什麼這需要寫 ZEP?你要解決什麼問題?如果沒有這條規則會怎樣?舉具體例子
(某個 bug、某個讓人困惑的 API、某次 onboarding 痛點)。

## 規範

實際的規則。要具體。需要時用程式碼樣本說明。涵蓋多條子規則時請用標題分節。

若是 `Standards` 類型的 ZEP,每條規則都必須**要嘛**由編譯器強制、**要嘛**可以
寫成 linter 規則。請說明屬於哪種。

## 設計理由(Rationale)

針對規範中每一個非顯而易見的決定,說明為什麼選這個方案而不是別的。這節是
未來的你(或想 Supersede 這份 ZEP 的下一份 ZEP)第一個會讀的。

## 向後相容性(Backwards Compatibility)

哪些現有程式碼會壞?如果有:怎麼遷移。如果沒有:明確寫出來,別讓審閱者猜。

## 參考實作

連到實作這份 ZEP 的 commit / 檔案。若是只把既有實務固化下來的
`Informational` ZEP,連到 stdlib 裡的代表例子。

## 待釐清問題(Open Questions)

本 ZEP **刻意不回答**的事。給未來想寫後續 ZEP 的貢獻者參考。

## 參考

外部連結:其他語言的前例、討論串、bug 回報。

## 版權

本文件置於公有領域。
