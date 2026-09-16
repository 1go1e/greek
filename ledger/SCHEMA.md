# SCHEMA.md — claims.csv 欄位規則

**唯讀**。變更須提案到 `reports/revisions/` 並更新 `CHANGELOG.md`。

| 欄 | 型 | 必填 | 說明 |
|---|---|---|---|
| `id` | int | ✓ | 自增，不重用 |
| `claim` | text | ✓ | 主張原文（一句話） |
| `topic` | text | ✓ | 主題分類（如：人物 / 年代 / 地理 / 思想 / 藝術） |
| `source_ref` | text | ✓ | 出處（DOI / ISBN / JSTOR ID / 博物館目錄 / URL），檢索當次回傳的結構化欄位 |
| `status` | enum | ✓ | `unverified` / `verified` / `partial` / `contested` / `unsupported` |
| `confidence` | float | ✓ | 0.0–1.0 |
| `evidence_note` | text |  | 檢索筆記（檢索式、檢索引擎、找到/沒找到的原因） |
| `checked_at` | date |  | 檢索日期（YYYY-MM-DD） |
| `checked_by` | text |  | 檢索者（pinger / wolf / other） |
| `supersedes` | int |  | 取代舊列的 id（判定改變時填） |
| `superseded_by` | int |  | 被新列取代的 id（被取代時填） |

## 寫入限制

- 不刪列。判定改變時新增一列，用 `supersedes` / `superseded_by` 標記關係。
- 識別碼不憑記憶寫。只填檢索工具當次回傳的結構化欄位。
- `confidence` 只在 `verified` / `partial` / `contested` 時有意義；`unverified` / `unsupported` 留 0.0。
- `evidence_note` 是審計軌跡。即使是 `unsupported` 也要記檢索過幾次、用什麼檢索式。
