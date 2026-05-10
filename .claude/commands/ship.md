---
description: 透過 parallel fan-out 給專家 personas 跑 pre-launch checklist，然後綜合成 go/no-go 決定。English: Run the pre-launch checklist via parallel fan-out to specialist personas, then synthesize a go/no-go decision
---

呼叫 agent-skills:shipping-and-launch skill。

`/ship` 是一個 **fan-out orchestrator**。它針對目前的變更平行跑三個專家 personas，然後把它們的報告合併為單一 go/no-go 決定，並附上 rollback plan。Personas 各自獨立運作 — 沒有共享 state、沒有順序 — 這正是 parallel execution 在這裡安全又有用的原因。

## Phase A — Parallel fan-out

用 Agent tool 同時 spawn 三個 subagents。**在同一個 assistant turn 中發出全部三個 Agent tool 呼叫，這樣它們才會 parallel 執行** — 依序呼叫會破壞此命令的目的。

在 Claude Code 中，每個呼叫傳入 `subagent_type` 對應到該 persona 的 `name` 欄位：

1. **`code-reviewer`** — 對 staged changes 或近期 commits 跑五面向 review（correctness、readability、architecture、security、performance）。輸出標準 review template。
2. **`security-auditor`** — 跑漏洞與威脅模型分析。檢查 OWASP Top 10、secrets handling、auth/authz、相依的 CVEs。輸出標準 audit report。
3. **`test-engineer`** — 分析該變更的 test 覆蓋率。找出 happy path、edge cases、error paths、以及 concurrency 情境的缺口。輸出標準 coverage analysis。

在沒有 Agent tool 的其他 harness 中，依序呼叫每個 persona 的 system prompt，並把它們的輸出當作 parallel 回傳處理 — 合併 phase 仍可運作。

限制（來自 Claude Code 的 subagent 模型）：
- Subagents 不能 spawn 其他 subagents — 不要讓某個 persona 委派給另一個。
- 每個 subagent 有自己的 context window，且只把報告回傳到這個主 session。
- 如果你需要彼此互通而不是只是回報的隊友，請使用 Claude Code Agent Teams，並把這些 personas 當成 teammate types 引用（見 `references/orchestration-patterns.md`）。

**Persona resolution。** 如果你已經在 `.claude/agents/` 或 `~/.claude/agents/` 自訂了 `code-reviewer`、`security-auditor` 或 `test-engineer`，那些優先於本 plugin 的版本 — `/ship` 會自動採用你的客製化。這是刻意設計的：plugin subagents 在 Claude Code 的 scope priority table 最底層，所以 user-level 定義依設計優先勝出。

## Phase B — Merge in main context

當三份報告都回來後，main agent（不是 sub-persona）綜合它們：

1. **Code Quality** — 匯總 `code-reviewer` 的 Critical/Important findings，以及任何失敗的 test、lint、或 build output。處理 reviewers 之間的重複項目。
2. **Security** — 把 `security-auditor` 的任何 Critical/High findings 提升為 launch blockers。與 `code-reviewer` 的 security 面向交叉比對。
3. **Performance** — 從 `code-reviewer` 的 performance 面向取得；如果適用，跨核 Core Web Vitals。
4. **Accessibility** — 驗證 keyboard nav、screen reader 支援、對比度（這三個 personas 不涵蓋 — 在這裡直接處理，或呼叫 accessibility checklist）。
5. **Infrastructure** — Env vars、migrations、monitoring、feature flags。直接驗證。
6. **Documentation** — README、ADRs、changelog。直接驗證。

## Phase C — Decision and rollback

產出單一輸出：

```markdown
## Ship Decision: GO | NO-GO

### Blockers (must fix before ship)
- [Source persona: Critical finding + file:line]

### Recommended fixes (should fix before ship)
- [Source persona: Important finding + file:line]

### Acknowledged risks (shipping anyway)
- [Risk + mitigation]

### Rollback plan
- Trigger conditions: [what signals would prompt rollback]
- Rollback procedure: [exact steps]
- Recovery time objective: [target]

### Specialist reports (full)
- [code-reviewer report]
- [security-auditor report]
- [test-engineer report]
```

## Rules

1. Phase A 的三個 personas 平行跑 — 絕不依序。
2. Personas 不互相呼叫。Main agent 在 Phase B 合併。
3. 任何 GO 決定之前，rollback plan 是必要的。
4. 如果任何 persona 回傳 Critical finding，預設 verdict 為 NO-GO，除非使用者明確接受該風險。
5. **只有在以下全部成立時才略過 fan-out：** 變更動到的檔案 ≤ 2 個、diff 在 50 行以內、且不觸及 auth、payments、data access、或 config/env。否則預設執行 fan-out。`/ship` 是為了即將上 production 的變更設計的 — 當 blast radius 不算微小時，即便 diff 看起來很小，也要跑 parallel review。
