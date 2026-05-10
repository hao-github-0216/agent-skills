---
description: 簡化 code 以提升清晰度與可維護性 — 在不改變行為的前提下降低複雜度。English: Simplify code for clarity and maintainability — reduce complexity without changing behavior
---

呼叫 agent-skills:code-simplification skill。

在保留完全相同行為的前提下，簡化最近變更的 code（或指定範圍）：

1. 讀 CLAUDE.md 並研究專案慣例
2. 鎖定目標 code — 預設是最近的變更，除非另有指定更廣的範圍
3. 在動手前先理解 code 的目的、呼叫者、edge cases、以及 test 覆蓋情況
4. 掃描可簡化的機會：
   - 深層 nesting → guard clauses 或抽出 helper
   - 過長 function → 依 responsibility 拆分
   - 巢狀 ternary → 改用 if/else 或 switch
   - 過於 generic 的命名 → 改用具描述性的名稱
   - 重複邏輯 → 抽成共用 function
   - Dead code → 確認後移除
5. incremental 套用每個簡化 — 每次變更後跑 test
6. 確認所有 test 通過、build 成功、diff 乾淨

如果簡化後 test 失敗，revert 該變更並重新評估。用 `code-review-and-quality` review 結果。
