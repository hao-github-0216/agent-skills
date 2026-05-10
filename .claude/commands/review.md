---
description: 進行五面向的 code review — correctness、readability、architecture、security、performance。English: Conduct a five-axis code review — correctness, readability, architecture, security, performance
---

呼叫 agent-skills:code-review-and-quality skill。

針對目前變更（已 staged 或近期 commits）跨五個面向 review：

1. **Correctness** — 是否符合 spec？edge cases 是否處理？test 是否充足？
2. **Readability** — 命名清楚嗎？邏輯直觀嗎？組織良好嗎？
3. **Architecture** — 是否遵循既有 patterns？邊界清晰？抽象層級合適？
4. **Security** — input 是否驗證？secrets 安全嗎？auth 是否檢查？（用 security-and-hardening skill）
5. **Performance** — 沒有 N+1 queries？沒有無上限的操作？（用 performance-optimization skill）

把發現分類為 Critical、Important、或 Suggestion。
輸出一份結構化的 review，附上具體的 file:line 引用與修正建議。
