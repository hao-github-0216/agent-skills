---
description: 跑 TDD workflow — 寫 failing test、實作、驗證。對 bug 採用 Prove-It pattern。English: Run TDD workflow — write failing tests, implement, verify. For bugs, use the Prove-It pattern.
---

呼叫 agent-skills:test-driven-development skill。

新功能：
1. 寫描述預期行為的 test（它們應該 FAIL）
2. 實作 code 讓它們通過
3. 在保持 test 綠燈的前提下 refactor

Bug 修正（Prove-It pattern）：
1. 寫一個能重現 bug 的 test（必須 FAIL）
2. 確認該 test 失敗
3. 實作修正
4. 確認該 test 通過
5. 跑完整 test suite 檢查 regression
