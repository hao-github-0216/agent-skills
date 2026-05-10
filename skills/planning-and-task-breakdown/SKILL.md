---
name: planning-and-task-breakdown
description: 將工作拆解為有順序的 task。當你已有 spec 或明確需求、需要把工作拆成可實作的單元時使用。當 task 感覺太大難以下手、需要估算 scope、或可以並行進行時也適用。English: Breaks work into ordered tasks. Use when you have a spec or clear requirements and need to break work into implementable tasks. Use when a task feels too large to start, when you need to estimate scope, or when parallel work is possible.
---

# Planning and Task Breakdown

## Overview

把工作拆解為小型、可驗證的 task，並附上明確的 acceptance criteria。良好的 task 拆解，是「能可靠完成工作的 agent」與「產出一團亂的 agent」之間的差別。每個 task 都應該小到能在一次專注的 session 內完成實作、測試與驗證。

## When to Use

- 你已有 spec，需要拆成可實作的單元
- 某個 task 感覺太大或太模糊難以下手
- 工作需要在多個 agent 或 session 間平行處理
- 你需要對人類傳達 scope
- 實作順序不明顯
- 你正在規劃多階段的 ML experiment campaign（sanity → ampere → dgxh）

**不適用的情況：** 單檔變更且 scope 很明顯，或 spec 已經包含定義良好的 task 時。

## ML experiment campaigns — partition escalation 即為依賴圖

對於 ML 研究的規劃，依賴圖不是「模組 A → 模組 B」，而是 **compute-tier escalation**。永遠先計畫便宜的 proof-of-life，再進入昂貴的 run：

```
Phase 0: SANITY (howardserver local, RTX 3060 Ti, ≤30 min, ≤8GB)
    │  Goal: code runs end-to-end, no exceptions, loss decreases for 100 steps
    │  Cost: free (local)
    │
    ▼
Phase 1: AMPERE (OSU HPC, A100 40/80GB, hours)
    │  Goal: full training run on real data + real batch size
    │  Cost: ~minutes-hours of A100 time
    │
    ▼
Phase 2: DGXH (OSU HPC, H100, only if measured >6h on A100 + cuDNN smoke OK)
       Goal: production-scale ablations
       Cost: scarce; requires explicit escalation justification
```

`dgxh200` partition 在這裡被禁用 — 永遠不要把它列入計畫。`preempt` 只適合 smoke run。

每個 phase 都是一個 checkpoint。**Phase N 失敗代表「不要往下推進」** — 先診斷並重新規劃。每個 phase 的 task 都應該標一行「expected wall time」，這樣才能判斷 job 是否卡住。

每個 Phase ≥1 的 task 都要在 submit 之前先建立一個 vault cell（`~/vault/<Project>/Experiments/<tag>.md`）。這個 cell 就是該 task 的 acceptance criteria 容器。

也可參考 `experiment-allocator` skill，依照各專案的 allocation rule 挑選正確的 partition。

## The Planning Process

### Step 1: 進入 Plan Mode

在寫任何 code 之前，以唯讀模式運作：

- 閱讀 spec 與相關 codebase 區段
- 找出既有的 patterns 與慣例
- 對應元件之間的依賴關係
- 記下風險與未知數

**規劃階段不要寫任何 code。** 此階段的產出是一份規劃文件，而不是實作。

### Step 2: 找出依賴圖

對應各個元件之間的依賴：

```
Database schema
    │
    ├── API models/types
    │       │
    │       ├── API endpoints
    │       │       │
    │       │       └── Frontend API client
    │       │               │
    │       │               └── UI components
    │       │
    │       └── Validation logic
    │
    └── Seed data / migrations
```

實作順序依照依賴圖由下往上：先打地基。

### Step 3: 垂直切分

不要先做完所有 database、再做完所有 API、再做完所有 UI — 而是一次完成「一條完整的功能路徑」：

**Bad (horizontal slicing):**
```
Task 1: Build entire database schema
Task 2: Build all API endpoints
Task 3: Build all UI components
Task 4: Connect everything
```

**Good (vertical slicing):**
```
Task 1: User can create an account (schema + API + UI for registration)
Task 2: User can log in (auth schema + API + UI for login)
Task 3: User can create a task (task schema + API + UI for creation)
Task 4: User can view task list (query + API + UI for list view)
```

每個垂直切片都會產出可運作、可測試的功能。

### Step 4: 撰寫 Task

每個 task 採用以下結構：

```markdown
## Task [N]: [Short descriptive title]

**Description:** One paragraph explaining what this task accomplishes.

**Acceptance criteria:**
- [ ] [Specific, testable condition]
- [ ] [Specific, testable condition]

**Verification:**
- [ ] Tests pass: `npm test -- --grep "feature-name"`
- [ ] Build succeeds: `npm run build`
- [ ] Manual check: [description of what to verify]

**Dependencies:** [Task numbers this depends on, or "None"]

**Files likely touched:**
- `src/path/to/file.ts`
- `tests/path/to/test.ts`

**Estimated scope:** [Small: 1-2 files | Medium: 3-5 files | Large: 5+ files]
```

### Step 5: 排序與設立 Checkpoint

安排 task，使其滿足：

1. 依賴關係已被滿足（先打地基）
2. 每個 task 完成後系統都處於可運作狀態
3. 每 2-3 個 task 之間設置驗證 checkpoint
4. 高風險的 task 排在前面（fail fast）

加入明確的 checkpoint：

```markdown
## Checkpoint: After Tasks 1-3
- [ ] All tests pass
- [ ] Application builds without errors
- [ ] Core user flow works end-to-end
- [ ] Review with human before proceeding
```

## Task Sizing Guidelines

| Size | Files | Scope | Example |
|------|-------|-------|---------|
| **XS** | 1 | 單一函式或 config 變更 | 新增一條 validation rule |
| **S** | 1-2 | 一個元件或 endpoint | 新增一個 API endpoint |
| **M** | 3-5 | 一個功能切片 | 使用者註冊流程 |
| **L** | 5-8 | 多元件功能 | 含篩選與分頁的搜尋 |
| **XL** | 8+ | **太大 — 應再拆細** | — |

如果一個 task 是 L 或更大，就應該拆成更小的 task。Agent 在 S 與 M 規模的 task 上表現最佳。

**何時要再拆細：**
- 需要超過一次專注 session 才能完成（大致超過 2 小時的 agent 工作量）
- 你無法以 3 點以內的 bullet 描述 acceptance criteria
- 它會碰到兩個以上彼此獨立的子系統（例如 auth 與 billing）
- 你發現自己在 task 標題裡寫「以及」（這是兩個 task 的徵兆）

## Plan Document Template

```markdown
# Implementation Plan: [Feature/Project Name]

## Overview
[One paragraph summary of what we're building]

## Architecture Decisions
- [Key decision 1 and rationale]
- [Key decision 2 and rationale]

## Task List

### Phase 1: Foundation
- [ ] Task 1: ...
- [ ] Task 2: ...

### Checkpoint: Foundation
- [ ] Tests pass, builds clean

### Phase 2: Core Features
- [ ] Task 3: ...
- [ ] Task 4: ...

### Checkpoint: Core Features
- [ ] End-to-end flow works

### Phase 3: Polish
- [ ] Task 5: ...
- [ ] Task 6: ...

### Checkpoint: Complete
- [ ] All acceptance criteria met
- [ ] Ready for review

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| [Risk] | [High/Med/Low] | [Strategy] |

## Open Questions
- [Question needing human input]
```

## Parallelization Opportunities

當有多個 agent 或 session 可用時：

- **可安全並行：** 互相獨立的功能切片、為已實作功能補上的測試、文件
- **必須循序執行：** Database migrations、共享狀態變更、依賴鏈
- **需要協調：** 共用同一份 API contract 的功能（先定義 contract，再並行）

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「邊做邊想就好」 | 這正是後來變成一團亂、需要重做的原因。10 分鐘的規劃可省下數小時。 |
| 「這些 task 很明顯」 | 還是寫下來。明確的 task 會浮現潛藏的依賴與被遺漏的邊界情境。 |
| 「規劃是額外負擔」 | 規劃就是工作本身。沒有計畫的實作只是在打字。 |
| 「我可以全部記在腦袋裡」 | Context window 是有限的。寫下來的計畫能跨越 session 邊界與 compaction。 |

## Red Flags

- 沒有寫下 task 列表就開始實作
- task 只寫「實作這個功能」卻沒有 acceptance criteria
- 計畫裡沒有任何驗證步驟
- 所有 task 都是 XL 規模
- task 之間沒有 checkpoint
- 沒有考慮依賴順序

## Verification

在開始實作之前，確認：

- [ ] 每個 task 都有 acceptance criteria
- [ ] 每個 task 都有驗證步驟
- [ ] task 依賴已被識別並正確排序
- [ ] 沒有任何 task 動到超過約 5 個檔案
- [ ] 主要 phase 之間都設有 checkpoint
- [ ] 人類已審閱並核准此計畫
