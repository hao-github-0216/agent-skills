---
description: incremental 實作下一個 task — build、test、verify、commit。English: Implement the next task incrementally — build, test, verify, commit
---

呼叫 agent-skills:incremental-implementation skill，搭配 agent-skills:test-driven-development。

從 plan 中挑出下一個待處理的 task。對每個 task：

1. 讀取該 task 的 acceptance criteria
2. 載入相關 context（既有 code、patterns、types）
3. 為預期行為寫一個 failing test（RED）
4. 實作能讓 test 通過的最少 code（GREEN）
5. 跑完整 test suite 檢查是否有 regression
6. 跑 build 確認可以編譯
7. 用具描述性的訊息 commit
8. 將該 task 標記為完成，再進到下一個

如果任何步驟失敗，依照 agent-skills:debugging-and-error-recovery skill 處理。
