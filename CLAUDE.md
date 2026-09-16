# CLAUDE.md — greek repo 規則

> 跟 `1go1e/glycocalyx` 同 pattern：blog → research → ledger。
> 來源: `https://1go1e.vivaldi.net/greek/`（2026-09-16）

## 規則

- **claims.csv 唯讀 append**：不刪列，判定改變用 supersede 欄。
- **識別碼只能來自檢索工具當次回傳**：DOI / ISBN / JSTOR ID / 博物館目錄號不憑記憶寫。
- **verified ≠ 共識**：verified = 「找到相符原始文獻」，不是「學界共識」、「歷史定論」或「臨床可用」。
- **原文不刪**：source/ 裡查不到的東西標註，不從原文刪掉。
- **Schema 變更**：先寫提案到 `reports/revisions/`，CHANGELOG.md 留紀錄。

## Status 定義（跟 glycocalyx 對齊）

| status | 意思 |
|---|---|
| unverified | 尚未檢索 |
| verified | 找到相符原始文獻 |
| partial | 方向 / 部分成立，但某個要素（年代、歸屬、人物）不成立 |
| contested | 找到的文獻彼此衝突，或對照組推翻該主張 |
| unsupported | 多次不同措辭檢索後仍查無出處。**unsupported 是合格結果** |

## 範圍

希臘文明 → 中亞 / 貴霜 / 大乘佛教擴散 — 從來源博客文拆出的歷史主張。
不做：醫療建議、現當代政治評斷、超出博客文範圍的擴張。

詳見 `STATUS.md`。
