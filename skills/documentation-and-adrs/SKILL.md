---
name: documentation-and-adrs
description: 記錄決策與文件。在做架構決策、變更 public API、出貨功能、或需要為未來工程師與 agent 留下理解 codebase 必要 context 時使用。English: Records decisions and documentation. Use when making architectural decisions, changing public APIs, shipping features, or when you need to record context that future engineers and agents will need to understand the codebase.
---

# Documentation and ADRs

## 概覽

要記錄的是決策，不只是 code。最有價值的文件捕捉的是 *為什麼* —— 導向某個決策的 context、限制、與權衡。Code 顯示 *做了什麼*；文件解釋 *為什麼這樣做* 與 *考慮過哪些 alternatives*。這個 context 對未來在 codebase 工作的人類與 agent 都至關重要。

## 何時使用

- 做重大架構決策時
- 在多個競爭方案中選擇時
- 新增或修改 public API 時
- 出貨會改變使用者面行為的功能時
- 讓新成員（或 agent）熟悉專案時
- 當你發現自己在重複解釋同一件事時

**何時不該用：**不要為顯而易見的 code 寫文件。不要寫只是把 code 重講一次的 comment。不要為一次性的原型寫文件。

## Architecture Decision Records (ADRs)

ADR 捕捉重大技術決策背後的推理過程。它是你能寫的文件中最高價值的一種。

### 何時寫 ADR

- 選 framework、library、或主要 dependency 時
- 設計 data model 或 database schema 時
- 選 authentication 策略時
- 決定 API 架構（REST vs. GraphQL vs. tRPC）時
- 在 build tool、hosting platform、或 infrastructure 之間選擇時
- 任何難以反轉的決策

### ADR Template

ADR 放在 `docs/decisions/`，依序編號：

```markdown
# ADR-001: Use PostgreSQL for primary database

## Status
Accepted | Superseded by ADR-XXX | Deprecated

## Date
2025-01-15

## Context
We need a primary database for the task management application. Key requirements:
- Relational data model (users, tasks, teams with relationships)
- ACID transactions for task state changes
- Support for full-text search on task content
- Managed hosting available (for small team, limited ops capacity)

## Decision
Use PostgreSQL with Prisma ORM.

## Alternatives Considered

### MongoDB
- Pros: Flexible schema, easy to start with
- Cons: Our data is inherently relational; would need to manage relationships manually
- Rejected: Relational data in a document store leads to complex joins or data duplication

### SQLite
- Pros: Zero configuration, embedded, fast for reads
- Cons: Limited concurrent write support, no managed hosting for production
- Rejected: Not suitable for multi-user web application in production

### MySQL
- Pros: Mature, widely supported
- Cons: PostgreSQL has better JSON support, full-text search, and ecosystem tooling
- Rejected: PostgreSQL is the better fit for our feature requirements

## Consequences
- Prisma provides type-safe database access and migration management
- We can use PostgreSQL's full-text search instead of adding Elasticsearch
- Team needs PostgreSQL knowledge (standard skill, low risk)
- Hosting on managed service (Supabase, Neon, or RDS)
```

### ADR 生命週期

```
PROPOSED → ACCEPTED → (SUPERSEDED or DEPRECATED)
```

- **不要刪舊 ADR。** 它們捕捉了歷史 context。
- 當決策改變時，寫一份新 ADR 引用並 supersede 舊的。

## Inline 文件

### 何時要寫 comment

寫 *why*，不要寫 *what*：

```typescript
// BAD: Restates the code
// Increment counter by 1
counter += 1;

// GOOD: Explains non-obvious intent
// Rate limit uses a sliding window — reset counter at window boundary,
// not on a fixed schedule, to prevent burst attacks at window edges
if (now - windowStart > WINDOW_SIZE_MS) {
  counter = 0;
  windowStart = now;
}
```

### 何時不要寫 comment

```typescript
// Don't comment self-explanatory code
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// Don't leave TODO comments for things you should just do now
// TODO: add error handling  ← Just add it

// Don't leave commented-out code
// const oldImplementation = () => { ... }  ← Delete it, git has history
```

### 文件化已知雷區

```typescript
/**
 * IMPORTANT: This function must be called before the first render.
 * If called after hydration, it causes a flash of unstyled content
 * because the theme context isn't available during SSR.
 *
 * See ADR-003 for the full design rationale.
 */
export function initializeTheme(theme: Theme): void {
  // ...
}
```

## API 文件

對 public API（REST、GraphQL、library interface）：

### 與 type 共置（TypeScript 推薦）

```typescript
/**
 * Creates a new task.
 *
 * @param input - Task creation data (title required, description optional)
 * @returns The created task with server-generated ID and timestamps
 * @throws {ValidationError} If title is empty or exceeds 200 characters
 * @throws {AuthenticationError} If the user is not authenticated
 *
 * @example
 * const task = await createTask({ title: 'Buy groceries' });
 * console.log(task.id); // "task_abc123"
 */
export async function createTask(input: CreateTaskInput): Promise<Task> {
  // ...
}
```

### REST API 用 OpenAPI / Swagger

```yaml
paths:
  /api/tasks:
    post:
      summary: Create a task
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskInput'
      responses:
        '201':
          description: Task created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
        '422':
          description: Validation error
```

## README 結構

每個專案都該有一份 README，涵蓋：

```markdown
# Project Name

One-paragraph description of what this project does.

## Quick Start
1. Clone the repo
2. Install dependencies: `npm install`
3. Set up environment: `cp .env.example .env`
4. Run the dev server: `npm run dev`

## Commands
| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server |
| `npm test` | Run tests |
| `npm run build` | Production build |
| `npm run lint` | Run linter |

## Architecture
Brief overview of the project structure and key design decisions.
Link to ADRs for details.

## Contributing
How to contribute, coding standards, PR process.
```

## Changelog 維護

對出貨功能：

```markdown
# Changelog

## [1.2.0] - 2025-01-20
### Added
- Task sharing: users can share tasks with team members (#123)
- Email notifications for task assignments (#124)

### Fixed
- Duplicate tasks appearing when rapidly clicking create button (#125)

### Changed
- Task list now loads 50 items per page (was 20) for better UX (#126)
```

## 給 Agent 看的文件

針對 AI agent context 的特別考量：

- **CLAUDE.md / rules 檔** —— 寫下專案慣例，讓 agent 跟著走
- **Spec 檔** —— 維持 spec 最新，讓 agent 做對的事
- **ADR** —— 幫 agent 理解過去決策的原因（避免重新決策）
- **Inline 雷區** —— 防止 agent 踩入已知陷阱

## 常見合理化（Rationalizations）

| 合理化說法 | 實際情況 |
|---|---|
| 「Code 就是文件」 | Code 顯示 what，不會顯示 why、考慮過什麼 alternative、有什麼限制。 |
| 「等 API 穩定了再寫文件」 | 寫了文件 API 才會更快穩定。文件是設計的第一個 test。 |
| 「沒人會看文件」 | Agent 會看，未來工程師會看，3 個月後的你也會看。 |
| 「ADR 是多餘負擔」 | 10 分鐘的 ADR 可以省下 6 個月後 2 小時的同題重辯。 |
| 「Comment 會過時」 | 寫 *why* 的 comment 是穩定的。會過時的是寫 *what* 的 comment —— 所以才只寫 why。 |

## Red Flags

- 架構決策沒有書面理由
- public API 沒有文件或 type
- README 沒寫怎麼跑專案
- 用 commented-out code 取代刪除
- TODO comment 放了好幾週
- 有重大架構選擇的專案卻沒有 ADR
- 文件只是把 code 重述一次，而不是解釋意圖

## 驗證

文件化後：

- [ ] 所有重大架構決策都有 ADR
- [ ] README 涵蓋 quick start、commands、與架構概觀
- [ ] API 函式有 parameter 與 return type 文件
- [ ] 重要的雷區在 inline 文件化在它出現的位置
- [ ] 沒有殘留的 commented-out code
- [ ] Rules 檔（CLAUDE.md 等）保持最新且準確
