---
name: code-simplification
description: 為了清晰度而簡化程式碼。用於不改變行為地 refactor 程式碼以提升可讀性時。用於程式碼能跑但比應有的更難讀、難維護、難擴充時。用於 review 累積了不必要複雜度的程式碼時。English: Simplifies code for clarity. Use when refactoring code for clarity without changing behavior. Use when code works but is harder to read, maintain, or extend than it should be. Use when reviewing code that has accumulated unnecessary complexity.
---

# Code Simplification

> Inspired by the [Claude Code Simplifier plugin](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md). Adapted here as a model-agnostic, process-driven skill for any AI coding agent.

## Overview

在精準保留行為的前提下降低複雜度來簡化程式碼。目標不是更少的行數——而是讓程式碼更容易閱讀、理解、修改與 debug。每次簡化都必須通過一個簡單測試：「新進團隊成員會比看原始碼更快理解嗎？」

## When to Use

- 功能可運作、測試通過後，但實作感覺比所需更笨重
- Code review 中提出可讀性或複雜度議題時
- 遇到深層巢狀邏輯、過長 function 或不清楚的命名時
- Refactor 在時間壓力下寫出的程式碼時
- 整合分散在多個檔案的相關邏輯時
- Merge 進帶來重複或不一致的變更後

**不該使用的時機：**

- 程式碼已經乾淨好讀——不要為簡化而簡化
- 你還不理解這段程式碼在做什麼——先理解再簡化
- 程式碼是 performance 關鍵段，「更簡單」的版本會明顯變慢
- 你即將整個改寫該 module——簡化拋棄式程式碼是浪費

## The Five Principles

### 1. Preserve Behavior Exactly

別改變程式碼做的事——只改它表達的方式。所有 input、output、side effect、錯誤行為與 edge case 都必須維持一致。如果你不確定一個簡化是否保留行為，就別做。

```
ASK BEFORE EVERY CHANGE:
→ Does this produce the same output for every input?
→ Does this maintain the same error behavior?
→ Does this preserve the same side effects and ordering?
→ Do all existing tests still pass without modification?
```

### 2. Follow Project Conventions

簡化的意思是讓程式碼更與 codebase 一致，而不是強加外來偏好。簡化前：

```
1. Read CLAUDE.md / project conventions
2. Study how neighboring code handles similar patterns
3. Match the project's style for:
   - Import ordering and module system
   - Function declaration style
   - Naming conventions
   - Error handling patterns
   - Type annotation depth
```

破壞專案一致性的「簡化」不是簡化——是無意義的擾動。

### 3. Prefer Clarity Over Cleverness

當壓縮版本需要停下來思考才能 parse 時，明白勝於精簡。

```typescript
// UNCLEAR: Dense ternary chain
const label = isNew ? 'New' : isUpdated ? 'Updated' : isArchived ? 'Archived' : 'Active';

// CLEAR: Readable mapping
function getStatusLabel(item: Item): string {
  if (item.isNew) return 'New';
  if (item.isUpdated) return 'Updated';
  if (item.isArchived) return 'Archived';
  return 'Active';
}
```

```typescript
// UNCLEAR: Chained reduces with inline logic
const result = items.reduce((acc, item) => ({
  ...acc,
  [item.id]: { ...acc[item.id], count: (acc[item.id]?.count ?? 0) + 1 }
}), {});

// CLEAR: Named intermediate step
const countById = new Map<string, number>();
for (const item of items) {
  countById.set(item.id, (countById.get(item.id) ?? 0) + 1);
}
```

### 4. Maintain Balance

簡化有一種失敗模式：過度簡化。注意以下陷阱：

- **過度 inline**——移除掉一個給概念取了名字的 helper 會讓呼叫端更難讀
- **混合不相關邏輯**——把兩個簡單 function 併成一個複雜 function 並不是簡化
- **移除「不必要」的 abstraction**——某些 abstraction 是為了擴展性或可測試性而存在，不是複雜度
- **以行數為優化目標**——更少的行數不是目標；更快的理解才是

### 5. Scope to What Changed

預設只簡化最近修改過的程式碼。除非被明確要求擴大範圍，避免順手 refactor 不相關的程式碼。範圍失控的簡化會讓 diff 變吵雜，並有非預期 regression 的風險。

## The Simplification Process

### Step 1: Understand Before Touching (Chesterton's Fence)

在改動或移除任何東西之前，先理解它為什麼存在。這就是 Chesterton's Fence：當你看到路上有道籬笆但不知道它為何在那，先別把它拆掉。先理解原因，再決定原因是否還適用。

```
BEFORE SIMPLIFYING, ANSWER:
- What is this code's responsibility?
- What calls it? What does it call?
- What are the edge cases and error paths?
- Are there tests that define the expected behavior?
- Why might it have been written this way? (Performance? Platform constraint? Historical reason?)
- Check git blame: what was the original context for this code?
```

如果回答不出來，你還沒準備好簡化。先讀更多上下文。

### Step 2: Identify Simplification Opportunities

掃描以下 pattern——每一個都是具體訊號，不是模糊感覺：

**結構複雜度：**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| 深層巢狀（3+ 層） | 控制流難以追蹤 | 抽出條件成 guard clause 或 helper function |
| 過長 function（50+ 行） | 多重職責 | 拆成具描述性命名的小 function |
| 巢狀 ternary | 需要心智 stack 才能 parse | 改成 if/else chain、switch 或查表物件 |
| Boolean 參數旗標 | `doThing(true, false, true)` | 改用 options object 或拆成不同 function |
| 重複的條件 | 相同 `if` 檢查出現多處 | 抽出成命名良好的 predicate function |

**命名與可讀性：**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| 通用命名 | `data`、`result`、`temp`、`val`、`item` | 改名描述內容：`userProfile`、`validationErrors` |
| 縮寫命名 | `usr`、`cfg`、`btn`、`evt` | 用完整字，除非縮寫是通用的（`id`、`url`、`api`） |
| 誤導命名 | 名為 `get` 的 function 卻會 mutate state | 改名以反映實際行為 |
| 解釋「what」的 comment | `count++` 上面寫 `// increment counter` | 刪掉 comment——程式碼已經夠清楚 |
| 解釋「why」的 comment | `// Retry because the API is flaky under load` | 保留——它承載程式碼無法表達的意圖 |

**冗餘：**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| 重複邏輯 | 相同 5+ 行出現多處 | 抽成共用 function |
| 死程式碼 | 到不了的分支、未用變數、註解掉的區塊 | 移除（先確認真的死了） |
| 不必要的 abstraction | 沒帶來價值的包裝 | Inline 包裝、直接呼叫底層 function |
| Over-engineer 的 pattern | factory-of-a-factory、只有一個 strategy 的 strategy | 改成簡單直接的作法 |
| 多餘的 type assertion | 對已經被推斷出來的型別做 cast | 移除 assertion |

### Step 3: Apply Changes Incrementally

一次做一個簡化。每次變更後跑測試。**Refactor 變更要與 feature 或 bug fix 變更分開送出。** 同時 refactor 又加 feature 的 PR 是兩個 PR——拆掉。

```
FOR EACH SIMPLIFICATION:
1. Make the change
2. Run the test suite
3. If tests pass → commit (or continue to next simplification)
4. If tests fail → revert and reconsider
```

避免把多個簡化打包成單一未測試的變更。如果壞掉，你需要知道是哪個簡化造成的。

**Rule of 500：** 如果一次 refactor 會動到超過 500 行，投資自動化（codemod、sed script、AST transform）而不是手動改。手動在這種規模下容易出錯，review 也累人。

### Step 4: Verify the Result

所有簡化做完後，退一步整體評估：

```
COMPARE BEFORE AND AFTER:
- Is the simplified version genuinely easier to understand?
- Did you introduce any new patterns inconsistent with the codebase?
- Is the diff clean and reviewable?
- Would a teammate approve this change?
```

如果「簡化過」的版本更難理解或更難 review，就 revert。並非每次簡化嘗試都會成功。

## Language-Specific Guidance

### TypeScript / JavaScript

```typescript
// SIMPLIFY: Unnecessary async wrapper
// Before
async function getUser(id: string): Promise<User> {
  return await userService.findById(id);
}
// After
function getUser(id: string): Promise<User> {
  return userService.findById(id);
}

// SIMPLIFY: Verbose conditional assignment
// Before
let displayName: string;
if (user.nickname) {
  displayName = user.nickname;
} else {
  displayName = user.fullName;
}
// After
const displayName = user.nickname || user.fullName;

// SIMPLIFY: Manual array building
// Before
const activeUsers: User[] = [];
for (const user of users) {
  if (user.isActive) {
    activeUsers.push(user);
  }
}
// After
const activeUsers = users.filter((user) => user.isActive);

// SIMPLIFY: Redundant boolean return
// Before
function isValid(input: string): boolean {
  if (input.length > 0 && input.length < 100) {
    return true;
  }
  return false;
}
// After
function isValid(input: string): boolean {
  return input.length > 0 && input.length < 100;
}
```

### Python

```python
# SIMPLIFY: Verbose dictionary building
# Before
result = {}
for item in items:
    result[item.id] = item.name
# After
result = {item.id: item.name for item in items}

# SIMPLIFY: Nested conditionals with early return
# Before
def process(data):
    if data is not None:
        if data.is_valid():
            if data.has_permission():
                return do_work(data)
            else:
                raise PermissionError("No permission")
        else:
            raise ValueError("Invalid data")
    else:
        raise TypeError("Data is None")
# After
def process(data):
    if data is None:
        raise TypeError("Data is None")
    if not data.is_valid():
        raise ValueError("Invalid data")
    if not data.has_permission():
        raise PermissionError("No permission")
    return do_work(data)
```

### React / JSX

```tsx
// SIMPLIFY: Verbose conditional rendering
// Before
function UserBadge({ user }: Props) {
  if (user.isAdmin) {
    return <Badge variant="admin">Admin</Badge>;
  } else {
    return <Badge variant="default">User</Badge>;
  }
}
// After
function UserBadge({ user }: Props) {
  const variant = user.isAdmin ? 'admin' : 'default';
  const label = user.isAdmin ? 'Admin' : 'User';
  return <Badge variant={variant}>{label}</Badge>;
}

// SIMPLIFY: Prop drilling through intermediate components
// Before — consider whether context or composition solves this better.
// This is a judgment call — flag it, don't auto-refactor.
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「能跑就不要動它」 | 能跑但難讀的程式碼，壞掉時也會難修。現在簡化能省下未來每次變更的時間。 |
| 「行數越少越簡單」 | 一行的巢狀 ternary 並不比五行的 if/else 簡單。簡化是關於理解速度，不是行數。 |
| 「順手把這段不相關的也簡化掉」 | 失控的簡化會產生吵雜的 diff、並讓你沒打算改的程式碼有 regression 風險。保持專注。 |
| 「type 已經自我文件化了」 | type 文件化結構、不是意圖。命名良好的 function 解釋「為什麼」遠勝過 type signature 解釋「是什麼」。 |
| 「這個 abstraction 之後可能用得到」 | 別保留投機性的 abstraction。如果現在沒在用，它就是沒帶價值的複雜度。先移除，需要時再加回。 |
| 「原作者一定有他的理由」 | 也許吧。看 git blame——套用 Chesterton's Fence。但累積的複雜度經常沒有理由；只是壓力下迭代留下的殘渣。 |
| 「我加 feature 的同時順便 refactor」 | 把 refactor 與 feature 分開。混合變更更難 review、revert 與在歷史中理解。 |

## Red Flags

- 需要修改測試才能通過的簡化（你很可能改了行為）
- 「簡化過」的程式碼比原版更長更難跟
- 為了配合自己的偏好而改名，而非依專案慣例
- 因為「程式碼變乾淨」就移除錯誤處理
- 簡化你還沒完全理解的程式碼
- 把多個簡化打包成一個大、難 review 的 commit
- 沒被要求就 refactor 任務範圍外的程式碼

## Verification

完成一次簡化後：

- [ ] 所有既有測試在不修改的情況下通過
- [ ] Build 成功且沒有新的 warning
- [ ] Linter/formatter 通過（沒有風格 regression）
- [ ] 每個簡化都是可 review 的漸進變更
- [ ] diff 乾淨——沒摻雜不相關的變更
- [ ] 簡化後的程式碼遵循專案慣例（對照 CLAUDE.md 或同等文件）
- [ ] 沒有錯誤處理被移除或弱化
- [ ] 沒有死程式碼遺留（未用的 import、到不了的分支）
- [ ] 隊友或 review agent 會把這個變更視為淨改善並 approve
