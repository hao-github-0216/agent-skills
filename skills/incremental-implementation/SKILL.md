---
name: incremental-implementation
description: 以 incremental 的方式交付變更。實作任何會動到一個檔案以上的功能或變更時使用。當你準備一次寫一大段程式碼，或一個任務感覺一步無法落地時使用。English: Delivers changes incrementally. Use when implementing any feature or change that touches more than one file. Use when you're about to write a large amount of code at once, or when a task feels too big to land in one step.
---

# Incremental Implementation

## Overview

以薄薄的縱向切片建構 — 實作一塊、測試它、驗證它，然後再擴張。避免一次實作整個 feature。每個 increment 應該讓系統處於可運作、可測試的狀態。這是讓大型 feature 變得可控的執行紀律。

## When to Use

- 實作任何跨檔案的變更
- 從任務分解開始打造一個新 feature
- refactor 既有程式碼
- 任何時候你想在測試之前寫超過約 100 行程式碼

**何時不要用：** 範圍已經極小的單檔、單函式變更。

## The Increment Cycle

```
┌──────────────────────────────────────┐
│                                      │
│   Implement ──→ Test ──→ Verify ──┐  │
│       ▲                           │  │
│       └───── Commit ◄─────────────┘  │
│              │                       │
│              ▼                       │
│          Next slice                  │
│                                      │
└──────────────────────────────────────┘
```

對每一個切片：

1. **Implement** 最小、完整的一塊功能
2. **Test** — 跑 test suite（如果沒有就寫一個 test）
3. **Verify** — 確認這個切片如預期運作（test 通過、build 成功、人工檢查）
4. **Commit** -- 用有描述性的 message 存進度（atomic commit 指引見 `git-workflow-and-versioning`）
5. **Move to the next slice** — 往前帶，不要重來

## Slicing Strategies

### Vertical Slices (Preferred)

打造一條穿過整個 stack 的完整路徑：

```
Slice 1: Create a task (DB + API + basic UI)
    → Tests pass, user can create a task via the UI

Slice 2: List tasks (query + API + UI)
    → Tests pass, user can see their tasks

Slice 3: Edit a task (update + API + UI)
    → Tests pass, user can modify tasks

Slice 4: Delete a task (delete + API + UI + confirmation)
    → Tests pass, full CRUD complete
```

每個切片都交付端到端可運作的功能。

### Contract-First Slicing

當 backend 跟 frontend 需要平行開發時：

```
Slice 0: Define the API contract (types, interfaces, OpenAPI spec)
Slice 1a: Implement backend against the contract + API tests
Slice 1b: Implement frontend against mock data matching the contract
Slice 2: Integrate and test end-to-end
```

### Risk-First Slicing

先處理風險最高、最不確定的那一塊：

```
Slice 1: Prove the WebSocket connection works (highest risk)
Slice 2: Build real-time task updates on the proven connection
Slice 3: Add offline support and reconnection
```

如果 Slice 1 失敗，你會在投入 Slice 2 和 3 之前就發現。

## Implementation Rules

### Rule 0: Simplicity First

寫任何程式碼之前，問自己：「能 work 的最簡作法是什麼？」

寫完之後，用以下檢查項目 review：
- 能不能用更少的行數做完？
- 這些抽象有付出對等的複雜度成本嗎？
- staff engineer 看到這段會不會說「為什麼不直接……」？
- 我是在為假設的未來需求建構，還是為當下的任務？

```
SIMPLICITY CHECK:
✗ Generic EventBus with middleware pipeline for one notification
✓ Simple function call

✗ Abstract factory pattern for two similar components
✓ Two straightforward components with shared utilities

✗ Config-driven form builder for three forms
✓ Three form components
```

三行類似的程式碼比過早抽象來得好。先實作天真、明顯正確的版本。在 test 證明正確性之後，才做最佳化。

### Rule 0.5: Scope Discipline

只動任務需要的東西。

不要：
- 「順手清掉」變更附近的程式碼
- 在沒在改的檔案裡 refactor import
- 移除你還不完全理解的註解
- 加上 spec 沒有、但「看起來有用」的 feature
- 在你只有讀的檔案裡把語法現代化

如果你發現任務範圍外有值得改善的東西，記下來 — 不要動手修：

```
NOTICED BUT NOT TOUCHING:
- src/utils/format.ts has an unused import (unrelated to this task)
- The auth middleware could use better error messages (separate task)
→ Want me to create tasks for these?
```

### Rule 1: One Thing at a Time

每個 increment 只改一件邏輯上的事。不要混淆 concern：

**Bad：** 一個 commit 加了一個新 component、refactor 了一個既有 component、又更新了 build config。

**Good：** 三個獨立的 commit — 每種變更各一個。

### Rule 2: Keep It Compilable

每個 increment 之後，專案必須能 build，既有 test 必須通過。不要在切片之間讓 codebase 處於壞掉的狀態。

### Rule 3: Feature Flags for Incomplete Features

如果一個 feature 還沒準備好給使用者，但你需要 merge 各個 increment：

```typescript
// Feature flag for work-in-progress
const ENABLE_TASK_SHARING = process.env.FEATURE_TASK_SHARING === 'true';

if (ENABLE_TASK_SHARING) {
  // New sharing UI
}
```

這讓你可以把小 increment merge 進 main branch，又不會把未完成的工作暴露出去。

### Rule 4: Safe Defaults

新程式碼預設應該採用安全、保守的行為：

```typescript
// Safe: disabled by default, opt-in
export function createTask(data: TaskInput, options?: { notify?: boolean }) {
  const shouldNotify = options?.notify ?? false;
  // ...
}
```

### Rule 5: Rollback-Friendly

每個 increment 應該可以獨立 revert：

- 加法式的變更（新檔案、新函式）容易 revert
- 對既有程式碼的修改應該最小、聚焦
- database migration 應該有對應的 rollback migration
- 避免在同一個 commit 裡刪掉一個東西又把它換掉 — 把這兩步分開

## Working with Agents

當你引導 agent 做 incremental 實作時：

```
"Let's implement Task 3 from the plan.

Start with just the database schema change and the API endpoint.
Don't touch the UI yet — we'll do that in the next increment.

After implementing, run `npm test` and `npm run build` to verify
nothing is broken."
```

對每個 increment 明確說明範圍是什麼、不是什麼。

## Increment Checklist

每個 increment 之後，驗證：

- [ ] 這個變更只做一件事，而且做完了
- [ ] 所有既有 test 仍然通過（`npm test`）
- [ ] build 成功（`npm run build`）
- [ ] type checking 通過（`npx tsc --noEmit`）
- [ ] linting 通過（`npm run lint`）
- [ ] 新功能如預期運作
- [ ] 變更已用有描述性的 message commit

**注意：** 在會影響該結果的變更後，跑對應的驗證指令。一次成功跑完之後，除非程式碼有改變，否則不要重複跑同一個指令 — 對沒變的程式碼重跑沒有提供任何資訊。

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「我最後再一起測試」 | bug 會疊加。Slice 1 的 bug 會讓 Slice 2-5 都錯。每個切片都要測。 |
| 「一次做完比較快」 | 直到出問題，而你找不到 500 行裡是哪一行造成的。當下感覺比較快而已。 |
| 「這些變更太小不值得分開 commit」 | 小 commit 是免費的。大 commit 藏 bug，rollback 也痛苦。 |
| 「feature flag 我之後再加」 | 如果 feature 沒做完，就不該對使用者可見。flag 現在就加。 |
| 「這個 refactor 小到可以順便」 | refactor 跟 feature 混在一起會讓兩邊都更難 review、更難 debug。分開。 |
| 「再跑一次 build 指令以防萬一」 | 一次成功跑完之後，重複同一個指令不會多給任何資訊，除非程式碼又改過。後續編輯後再跑，不要因為求心安而跑。 |

## Red Flags

- 在沒跑 test 的情況下寫了超過 100 行程式碼
- 在單一個 increment 裡放了多個無關的變更
- 「順便快速加上這個」式的 scope 擴張
- 為了趕進度跳過 test/verify 步驟
- increment 之間 build 或 test 壞掉
- 大量未 commit 的變更累積中
- 在第三個 use case 出現之前就建構抽象
- 在任務範圍外動檔案，理由是「我都已經來了」
- 為一次性操作建立新的 utility 檔案
- 在沒有任何中間程式碼變動的情況下，連跑兩次同樣的 build/test 指令

## Verification

完成一個任務的所有 increment 之後：

- [ ] 每個 increment 都個別測試並 commit 過
- [ ] 完整 test suite 通過
- [ ] build 乾淨
- [ ] feature 端到端如 spec 運作
- [ ] 沒有未 commit 的變更殘留
