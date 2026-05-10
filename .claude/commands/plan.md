---
description: 將工作拆解為小而可驗證的 task，附上 acceptance criteria 與相依順序。English: Break work into small verifiable tasks with acceptance criteria and dependency ordering
---

呼叫 agent-skills:planning-and-task-breakdown skill。

讀既有的 spec（SPEC.md 或同等文件）以及相關的 codebase 區段。然後：

1. 進入 plan mode — 只讀，不改 code
2. 找出元件之間的相依圖
3. 垂直切分工作（每個 task 是一條完整路徑，而非水平 layer）
4. 寫出 task，附上 acceptance criteria 與驗證步驟
5. 在各 phase 之間加上 checkpoint
6. 將 plan 提交給人類 review

把 plan 存到 tasks/plan.md，task list 存到 tasks/todo.md。
