# plan.md — greek repo pipeline 設計

> 從 `koan.vivaldi.net/greek` 博客文 → evidence-driven repo 的 pipeline 設計文件。
> 討論時段: 2026-09-16 22:40–23:10 GMT+9（pinger 🍐 + wolf 🐺）
> 後續 session 從這份文件接, runbooks/ 依 §9 順序寫。

---

## 1. Pipeline（8 stage + 1 gate）

```
BLOG URL
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. INTAKE  (subagent, 1 model call)                         │
│    Input:  blog URL → web_fetch → raw text                  │
│    Process: LLM extracts claims（1 call）                   │
│    Output: staging/<batch>/claims-raw.md                    │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
[orchestrator: split by topic, no LLM]
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. RESEARCHER  (subagent, N serial calls, NOT parallel)     │
│    For each claim i in 1..N:                                │
│      Input:  claim_i + topic_i                              │
│      Process: web_search ×1-3 + LLM synthesis（1 call）      │
│      Output: append to staging/<batch>/evidence-<topic>.md   │
│    Total: N model calls（1 in-flight at a time）             │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. VERIFIER  (subagent, N serial calls)                      │
│    For each claim i:                                         │
│      Input:  claim_i + evidence_i                           │
│      Process: LLM match（1 call）→ status + confidence       │
│      Output: append to staging/<batch>/verification.md       │
│    Total: N model calls                                      │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 3.5 CONSISTENCY-CHECK  (gate, 1 model call / batch)          │
│    Input:  staging/<batch>/verification.md（將進 ledger 的）  │
│    Process: 1 LLM call 掃:                                  │
│      - 每列 source_ref 或 evidence_note 都有?                │
│      - status 分佈合理?（不會 100% verified 或 100% unsupported）│
│      - 有無可疑模式?（同 source 重複用 / 同 claim 反覆出現）  │
│    Output: staging/<batch>/consistency.md（pass / fail + 修正清單）│
│    Fail → 回到 researcher 或 verifier 修, 不進 ledger-writer │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. LEDGER-WRITER  (subagent, 0 model calls, just CSV)        │
│    Input:  verification.md（consistency-check pass）          │
│    Process: parse + append row to claims.csv                 │
│    Output: ledger/claims.csv (append-only)                   │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
[orchestrator: compute diff for report, no LLM]
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. REPORTER  (subagent, 1 model call)                        │
│    Input:  claims.csv diff + verification notes              │
│    Process: LLM batch summary（1 call）                      │
│    Output: reports/backfill/<batch>.md                       │
└─────────────────────────────────────────────────────────────┘
  │
  ▼
[orchestrator: human review gate]
  │   wolf 看 report.md + claims.csv diff → approve 才 push
  ▼
[orchestrator: git commit + push, 0 model calls]
  │
  ▼
GITHUB updated
```

### 1.1 為何不平行 topic subagents

- 4 個 topic subagent 同時跑 = 撞 rate limit + 排隊 latency + USD cost
- Latency: 平行可能不比 serial 快（rate limit queue）
- Memory/context bloat: 每 subagent fork 一份 context
- Output quality: 平行容易 verification 互相干擾
- Debugging hell: 失敗時難定位
- 改成全 serial: 同時間永遠只有 1 個 model call in-flight

### 1.2 Model call budget per batch（N=10）

| Stage | calls | in-flight max |
|---|---|---|
| intake | 1 | 1 |
| researcher | 10 | **1**（serial） |
| verifier | 10 | **1**（serial） |
| consistency-check | 1 | 1 |
| ledger-writer | 0 | 0 |
| reporter | 1 | 1 |
| publish | 0 | 0 |
| **total** | **23** | **1** |

---

## 2. 切工作（雙軸）

### 2.1 Batch 軸

- 一次處理 N=8–10 條 claim
- Bounded, 進度可追蹤, 失敗時損失小
- Batch 命名: `Batch 01`, `Batch 02`...

### 2.2 Topic 軸（researcher 內部切）

- claim 依 topic 分: 人物 / 年代 / 地理 / 思想 / 藝術 / 文本 / 政治經濟
- Topic 在 **researcher 內部** 切（同一 subagent 內 serial）, 不是 subagent 間 parallel
- 每 topic 寫到獨立 file: `evidence-<topic>.md`

---

## 3. 固定 subagent（5 個）

| 角色 | 輸入 | 輸出 | runbook |
|---|---|---|---|
| **intake** | blog URL | staging/<batch>/claims-raw.md | `runbooks/intake.md` |
| **researcher** | claim + topic | staging/<batch>/evidence-<topic>.md | `runbooks/research.md` |
| **verifier** | claim + evidence | staging/<batch>/verification.md | `runbooks/verify.md` |
| **consistency-check** | verification.md | staging/<batch>/consistency.md | `runbooks/consistency-check.md` |
| **reporter** | claims.csv diff | reports/backfill/<batch>.md | `runbooks/reporter.md` |

`ledger-writer` 是純 text editor, 不需 LLM — orchestrator 直接做, 或寫成 bash script。
`publisher` 也一樣 — orchestrator 跑 `git add/commit/push`。

### 3.1 Session / Lifecycle

```
orchestrator (pinger main, persistent)
  │
  ├─ spawn intake           (one-shot, exit after output)
  ├─ spawn researcher       (one-shot, exit after N claims done)
  ├─ spawn verifier         (one-shot, exit after N claims done)
  ├─ spawn consistency-check (one-shot, exit after pass/fail)
  ├─ spawn reporter         (one-shot, exit after report.md)
  │
  └─ (inline) ledger-writer + publisher (orchestrator 自己跑)

每 subagent:
  - context: forked from orchestrator（拿 CLAUDE.md + 對應 runbook）
  - 工作目錄: greek/
  - 寫入: staging/<batch>/ (own scope, 不碰 ledger/ 跟 reports/)
  - 結束: 回報 orchestrator + exit
```

---

## 4. Model 選擇

| 用途 | 模型 | 理由 |
|---|---|---|
| intake / verifier / reporter / consistency-check | `minimax-portal/MiniMax-M3` (primary) | flat rate, 不燒 USD |
| researcher LLM synthesis | `minimax-portal/MiniMax-M3` (primary) | 同上, 一致性 |
| Fallback (rate-limit hit) | `deepseek/deepseek-v4-pro` | 從 `openclaw.json` 已配, subagent prompt 寫切換 SOP |
| web_search / web_fetch | n/a | Tailscale / GitHub 公共 API, 無 model call |

---

## 5. Checkpoint

### 5.1 File-based（auto）

每 stage 寫完 disk file = 自然 checkpoint:

| Stage | File |
|---|---|
| intake | `staging/<batch>/claims-raw.md` |
| researcher | `staging/<batch>/evidence-<topic>.md` (per topic) |
| verifier | `staging/<batch>/verification.md` |
| consistency-check | `staging/<batch>/consistency.md` |
| ledger-writer | `ledger/claims.csv` |
| reporter | `reports/backfill/<batch>.md` |

Crash recovery = 讀 file + 從中斷處 resume。

### 5.2 Human review gate

推到 GitHub **前**: wolf 看 `reports/backfill/<batch>.md` + `ledger/claims.csv` diff → approve 才 push。

---

## 6. Push back

| 層 | 機制 |
|---|---|
| Claim 改判定 | `superseded_by` 欄 — 舊列保留, 新列填 supersede id, 不刪 |
| Wolf 推翻 verifier | 直接 edit `claims.csv` (或寫 "wolf-override" runbook 把 verdict 寫進 evidence_note, 不動 status 欄) |
| Schema 變更 | **不能直接改 SCHEMA.md** — 寫提案到 `reports/revisions/`, 更新 `CHANGELOG.md`, apply |
| Batch 整批退 | push 前 `git reset`; push 後 revert commit + 補修正 commit |
| Subagent 亂寫 staging/ | consistency-check 抓到 → 回到 researcher / verifier 修 |
| Model 亂講（hallucination） | supersede + evidence_note 記 |

---

## 7. Audit

**不要專職 audit subagent** — 理由:
- verifier 已經做 claim ↔ source cross-check（核心 audit）
- 推 GitHub 前 wolf review（人類 audit）
- 多 1 個 audit subagent = 多 N 個 model call per batch, 撞 rate limit + 燒 USD
- Latency

**替代**: `consistency-check` 階段（§3 stage 3.5）, 1 model call / batch, gate 而非 audit。

---

## 8. Flow topology

**1 條垂直線, serial, no branching, no parallel。**

- 不同 batch 不同時間跑（不是平行）
- 不同 repo（greek / glycocalyx / future）各自一份 SOP + ledger, 不共用 pipeline
- 多 source 跨語言（中文 + 英文 + 梵文）是 researcher 內部 sub-call, 不是 pipeline-level parallel

---

## 9. Runbook 寫作順序

1. `runbooks/00-pipeline.md` — 本文件的執行版（含所有 §細節）
2. `runbooks/intake.md`
3. `runbooks/research.md`
4. `runbooks/verify.md`
5. `runbooks/consistency-check.md`
6. `runbooks/ledger.md` (ledger-writer SOP, 純 text editing 不需 LLM)
7. `runbooks/reporter.md`
8. `runbooks/publish.md` (optional, git SOP)

每個 runbook 統一格式:
```
# 角色 (一句話)
# 輸入
# 輸出
# 程序（step 1, step 2, ...）
# 失敗 / 例外
# 驗證
```

---

## 10. Source / Reference / State

- **Source blog**: `https://1go1e.vivaldi.net/greek/`（"希臘文明與貴霜王朝" 2026-09-16, ~16K chars）
- **Reference repo**: `https://github.com/1go1e/glycocalyx` (同 pattern, 中文 medical evidence-driven)
- **Local**: `~/osaka_obsidian/1go1e_pinger/greek/`
- **Deploy key**: `~/.ssh/1go1e_greek_deploy_ed25519` (ed25519, wolf 已貼 GitHub)
- **Repo state**: 已 init commit
  - `CLAUDE.md` (1308B) — 規則
  - `STATUS.md` (710B) — 入口
  - `README.md` (762B)
  - `ledger/SCHEMA.md` (1421B) — 11 欄定義
  - `ledger/claims.csv` (105B, header only, 0 列)
  - `.gitkeep` × 5 (空 dir)

---

## 11. Next session 任務

1. 讀 `plan.md` (本文件) 拿 context
2. 從 `runbooks/00-pipeline.md` 開始寫（把 §1–§8 細化成可執行 SOP）
3. 然後 `runbooks/intake.md` → `research.md` → `verify.md` → ...
4. 第一個 batch: Batch 01 dry-run, 用 `koan.vivaldi.net/greek`
5. 失敗修 SOP, 成功 push first real commit
