---
name: test-driven-development
description: 用 test 驅動開發。實作任何邏輯、修任何 bug、改任何行為時都使用。當你需要證明 code 真的能跑、收到 bug report、或即將修改既有功能時使用。English: Drives development with tests. Use when implementing any logic, fixing any bug, or changing any behavior. Use when you need to prove that code works, when a bug report arrives, or when you're about to modify existing functionality.
---

# Test-Driven Development

## 概覽

在寫讓 test 通過的 code 之前，先寫一個會 fail 的 test。修 bug 時，先用 test 重現 bug 再嘗試修。Test 是證據 ——「看起來對」不算完成。Test 寫得好的 codebase 是 AI agent 的超能力；沒有 test 的 codebase 則是負債。

## 何時使用

- 實作任何新邏輯或行為
- 修任何 bug（Prove-It Pattern）
- 修改既有功能
- 加入 edge case 處理
- 任何可能破壞既有行為的變更

**何時不用：** 純設定變更、文件更新，或對行為毫無影響的靜態內容變更。

**相關：** 涉及瀏覽器的變更，請把 TDD 結合 Chrome DevTools MCP 做執行期驗證 —— 見下方 Browser Testing 段落。

## TDD Cycle

```
    RED                GREEN              REFACTOR
 Write a test    Write minimal code    Clean up the
 that fails  ──→  to make it pass  ──→  implementation  ──→  (repeat)
      │                  │                    │
      ▼                  ▼                    ▼
   Test FAILS        Test PASSES         Tests still PASS
```

### Step 1：RED —— 寫一個會 fail 的 test

先寫 test，它必須 fail。一個一開始就 pass 的 test 什麼都沒證明。

```typescript
// RED: This test fails because createTask doesn't exist yet
describe('TaskService', () => {
  it('creates a task with title and default status', async () => {
    const task = await taskService.createTask({ title: 'Buy groceries' });

    expect(task.id).toBeDefined();
    expect(task.title).toBe('Buy groceries');
    expect(task.status).toBe('pending');
    expect(task.createdAt).toBeInstanceOf(Date);
  });
});
```

### Step 2：GREEN —— 讓它過

寫最少的 code 讓 test 通過。不要過度設計：

```typescript
// GREEN: Minimal implementation
export async function createTask(input: { title: string }): Promise<Task> {
  const task = {
    id: generateId(),
    title: input.title,
    status: 'pending' as const,
    createdAt: new Date(),
  };
  await db.tasks.insert(task);
  return task;
}
```

### Step 3：REFACTOR —— 整理乾淨

Test 都綠了之後，在不改變行為的前提下改進 code：

- 抽出共用邏輯
- 改善命名
- 移除重複
- 必要時優化

每做一步 refactor 就跑 test，確認沒有東西壞掉。

## Prove-It Pattern（修 bug）

收到 bug report 時，**不要直接動手修。** 先寫一個重現它的 test。

```
Bug report arrives
       │
       ▼
  Write a test that demonstrates the bug
       │
       ▼
  Test FAILS (confirming the bug exists)
       │
       ▼
  Implement the fix
       │
       ▼
  Test PASSES (proving the fix works)
       │
       ▼
  Run full test suite (no regressions)
```

**範例：**

```typescript
// Bug: "Completing a task doesn't update the completedAt timestamp"

// Step 1: Write the reproduction test (it should FAIL)
it('sets completedAt when task is completed', async () => {
  const task = await taskService.createTask({ title: 'Test' });
  const completed = await taskService.completeTask(task.id);

  expect(completed.status).toBe('completed');
  expect(completed.completedAt).toBeInstanceOf(Date);  // This fails → bug confirmed
});

// Step 2: Fix the bug
export async function completeTask(id: string): Promise<Task> {
  return db.tasks.update(id, {
    status: 'completed',
    completedAt: new Date(),  // This was missing
  });
}

// Step 3: Test passes → bug fixed, regression guarded
```

## Test Pyramid

依 pyramid 配置 test 投資 —— 大多數 test 應該又小又快，越往高層 test 越少：

```
          ╱╲
         ╱  ╲         E2E Tests (~5%)
        ╱    ╲        Full user flows, real browser
       ╱──────╲
      ╱        ╲      Integration Tests (~15%)
     ╱          ╲     Component interactions, API boundaries
    ╱────────────╲
   ╱              ╲   Unit Tests (~80%)
  ╱                ╲  Pure logic, isolated, milliseconds each
 ╱──────────────────╲
```

**Beyonce 法則：** If you liked it, you should have put a test on it. 基礎建設變更、refactor、migration 沒有義務替你抓 bug —— 你的 test 才有。如果一個變更弄壞了你的 code 而你沒有 test 守著，那是你的責任。

### Test Sizes（資源模型）

除了 pyramid 層級之外，再依 test 消耗的資源分類：

| Size | Constraints | Speed | Example |
|------|------------|-------|---------|
| **Small** | Single process, no I/O, no network, no database | Milliseconds | Pure function tests, data transforms |
| **Medium** | Multi-process OK, localhost only, no external services | Seconds | API tests with test DB, component tests |
| **Large** | Multi-machine OK, external services allowed | Minutes | E2E tests, performance benchmarks, staging integration |

Small test 應該佔測試套件絕大多數。它們快、穩定，fail 時也容易 debug。

### 決策指南

```
Is it pure logic with no side effects?
  → Unit test (small)

Does it cross a boundary (API, database, file system)?
  → Integration test (medium)

Is it a critical user flow that must work end-to-end?
  → E2E test (large) — limit these to critical paths
```

## 寫好 test

### Test state，不要 test interaction

斷言操作的*結果*，不要斷言內部呼叫了哪些 method。驗證 method 呼叫順序的 test，在你 refactor 時就會 break，即使行為沒變。

```typescript
// Good: Tests what the function does (state-based)
it('returns tasks sorted by creation date, newest first', async () => {
  const tasks = await listTasks({ sortBy: 'createdAt', sortOrder: 'desc' });
  expect(tasks[0].createdAt.getTime())
    .toBeGreaterThan(tasks[1].createdAt.getTime());
});

// Bad: Tests how the function works internally (interaction-based)
it('calls db.query with ORDER BY created_at DESC', async () => {
  await listTasks({ sortBy: 'createdAt', sortOrder: 'desc' });
  expect(db.query).toHaveBeenCalledWith(
    expect.stringContaining('ORDER BY created_at DESC')
  );
});
```

### Test 中 DAMP 優於 DRY

在 production code 中，DRY（Don't Repeat Yourself）通常是對的。在 test 中則是 **DAMP（Descriptive And Meaningful Phrases）** 比較好。Test 應該讀起來像一份規格 —— 每個 test 應該獨立講完一個故事，不需要讀者去追蹤共用 helper。

```typescript
// DAMP: Each test is self-contained and readable
it('rejects tasks with empty titles', () => {
  const input = { title: '', assignee: 'user-1' };
  expect(() => createTask(input)).toThrow('Title is required');
});

it('trims whitespace from titles', () => {
  const input = { title: '  Buy groceries  ', assignee: 'user-1' };
  const task = createTask(input);
  expect(task.title).toBe('Buy groceries');
});

// Over-DRY: Shared setup obscures what each test actually verifies
// (Don't do this just to avoid repeating the input shape)
```

只要能讓每個 test 獨立易懂，test 中的重複是可以接受的。

### 偏好真實實作而非 mock

挑能完成任務的最簡單 test double。Test 用越多真實 code，能給的信心越高。

```
Preference order (most to least preferred):
1. Real implementation  → Highest confidence, catches real bugs
2. Fake                 → In-memory version of a dependency (e.g., fake DB)
3. Stub                 → Returns canned data, no behavior
4. Mock (interaction)   → Verifies method calls — use sparingly
```

**只有以下情況才使用 mock：** 真實實作太慢、不確定、或副作用無法控制（外部 API、寄信）。Over-mock 會造成 test 過了但 production 壞掉。

### 使用 Arrange-Act-Assert pattern

```typescript
it('marks overdue tasks when deadline has passed', () => {
  // Arrange: Set up the test scenario
  const task = createTask({
    title: 'Test',
    deadline: new Date('2025-01-01'),
  });

  // Act: Perform the action being tested
  const result = checkOverdue(task, new Date('2025-01-02'));

  // Assert: Verify the outcome
  expect(result.isOverdue).toBe(true);
});
```

### 一個 test 只驗一個概念

```typescript
// Good: Each test verifies one behavior
it('rejects empty titles', () => { ... });
it('trims whitespace from titles', () => { ... });
it('enforces maximum title length', () => { ... });

// Bad: Everything in one test
it('validates titles correctly', () => {
  expect(() => createTask({ title: '' })).toThrow();
  expect(createTask({ title: '  hello  ' }).title).toBe('hello');
  expect(() => createTask({ title: 'a'.repeat(256) })).toThrow();
});
```

### Test 要命名得有描述性

```typescript
// Good: Reads like a specification
describe('TaskService.completeTask', () => {
  it('sets status to completed and records timestamp', ...);
  it('throws NotFoundError for non-existent task', ...);
  it('is idempotent — completing an already-completed task is a no-op', ...);
  it('sends notification to task assignee', ...);
});

// Bad: Vague names
describe('TaskService', () => {
  it('works', ...);
  it('handles errors', ...);
  it('test 3', ...);
});
```

## 要避免的 test 反模式

| Anti-Pattern | Problem | Fix |
|---|---|---|
| 測試實作細節 | Refactor 即使行為沒變 test 也 break | Test input 與 output，不 test 內部結構 |
| Flaky test（時序、依賴順序） | 削弱對測試套件的信任 | 使用確定性斷言、隔離 test state |
| 測試 framework code | 浪費時間 test 第三方行為 | 只 test 你寫的 code |
| Snapshot 濫用 | 大段沒人審的 snapshot，任何變更就 break | 節制使用 snapshot，每次變更都審 |
| 沒有 test 隔離 | 個別跑過、一起跑就 fail | 每個 test 自己 setup/teardown 自己的 state |
| 什麼都 mock | Test 過但 production 壞 | 偏好順序：real > fake > stub > mock。只在 boundary 處 mock，且只在真實依賴慢或不確定時 |

## 何時用 subagent 跑測試

複雜的 bug 修復可以開 subagent 寫重現 test：

```
Main agent: "Spawn a subagent to write a test that reproduces this bug:
[bug description]. The test should fail with the current code."

Subagent: Writes the reproduction test

Main agent: Verifies the test fails, then implements the fix,
then verifies the test passes.
```

這種分工確保 test 是在不知道 fix 內容的情況下寫出來的，可靠度更高。

## 延伸閱讀

跨 framework 的詳細 testing pattern、範例與 anti-pattern 見 `references/testing-patterns.md`。

## 常見的合理化說法

| Rationalization | Reality |
|---|---|
| 「我寫完 code 再補 test」 | 你不會的。事後補的 test 是測 implementation 不是行為。 |
| 「這太簡單不用 test」 | 簡單的 code 會變複雜。Test 文件化預期行為。 |
| 「Test 拖慢我」 | 現在拖慢。但每次後續改 code 都會幫你加速。 |
| 「我手動測過了」 | 手動測試不會留下來。明天的變更 break 它你也不會知道。 |
| 「Code 自我說明」 | Test **就是**規格。它記錄 code 應該做什麼，不是它現在做什麼。 |
| 「只是 prototype」 | Prototype 會變 production code。第一天就有 test 才能避免「test debt」危機。 |
| 「再跑一次 test 確認一下好了」 | 一次乾淨的 test run 之後，沒改任何 code 就重跑同個指令毫無意義。改完之後再跑，不要拿來自我安慰。 |

## Red Flags

- 寫 code 但完全沒有對應 test
- Test 第一次就 pass（它可能根本沒在測你以為的東西）
- 「全部 test pass」但其實根本沒跑 test
- 修 bug 沒附重現 test
- Test 在測 framework 行為而不是 application 行為
- Test 名稱沒有描述預期行為
- 跳過 test 來讓套件通過
- 沒改任何 code 就連續跑同個 test 指令兩次

## Verification

完成任何實作後：

- [ ] 每個新行為都有對應 test
- [ ] 全部 test pass：`npm test`
- [ ] Bug fix 都附了一個在 fix 之前會 fail 的重現 test
- [ ] Test 名稱描述了被驗證的行為
- [ ] 沒有任何 test 被 skip 或 disable
- [ ] Coverage 沒有下降（如果有追蹤）

**Note：** 每次做了可能影響結果的變更後就跑對應的 test 指令。乾淨跑過之後，code 沒改就不要重跑同個指令 —— 對沒變更的 code 重跑不會增加任何信心。
