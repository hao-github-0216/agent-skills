---
name: api-and-interface-design
description: 引導穩定的 API 與 interface 設計。用於設計 API、模組邊界或任何 public interface 時。用於建立 REST 或 GraphQL endpoint、定義模組之間的 type contract，或建立 frontend 與 backend 之間邊界時。English: Guides stable API and interface design. Use when designing APIs, module boundaries, or any public interface. Use when creating REST or GraphQL endpoints, defining type contracts between modules, or establishing boundaries between frontend and backend.
---

# API and Interface Design

## Overview

設計穩定、文件完善且難以誤用的 interface。良好的 interface 讓正確的事情變得容易、錯誤的事情變得困難。這適用於 REST API、GraphQL schema、模組邊界、component props，以及任何一段程式碼跟另一段對話的介面。

## When to Use

- 設計新的 API endpoint
- 定義模組邊界或團隊之間的 contract
- 建立 component prop interface
- 確立會影響 API 形狀的 database schema
- 變更現有的 public interface

## Core Principles

### Hyrum's Law

> With a sufficient number of users of an API, all observable behaviors of your system will be depended on by somebody, regardless of what you promise in the contract.

意思是：每個可觀察的行為——包含未文件化的怪癖、錯誤訊息文字、時序與排序——一旦有使用者依賴它，就成為事實上的 contract。設計上的意涵：

- **對於暴露的東西要有意識。** 每個可觀察的行為都是一個潛在的承諾。
- **不要洩漏實作細節。** 如果使用者觀察得到，他們就會依賴它。
- **在設計階段就規劃 deprecation。** 參見 `deprecation-and-migration`，了解如何安全地移除使用者所依賴的東西。
- **單靠測試不夠。** 即使有完美的 contract test，Hyrum's Law 也意味著「安全」的變更仍可能讓依賴未文件化行為的真實使用者出問題。

### The One-Version Rule

避免迫使 consumer 在同一個 dependency 或 API 的多個版本之間做選擇。當不同 consumer 需要同一個東西的不同版本時，就會產生 diamond dependency 問題。設計時假設同一時間只有一個版本存在——擴充而不要分叉。

### 1. Contract First

在實作之前先定義 interface。Contract 就是 spec——實作隨之而來。

```typescript
// Define the contract first
interface TaskAPI {
  // Creates a task and returns the created task with server-generated fields
  createTask(input: CreateTaskInput): Promise<Task>;

  // Returns paginated tasks matching filters
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;

  // Returns a single task or throws NotFoundError
  getTask(id: string): Promise<Task>;

  // Partial update — only provided fields change
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;

  // Idempotent delete — succeeds even if already deleted
  deleteTask(id: string): Promise<void>;
}
```

### 2. Consistent Error Semantics

選一種錯誤策略並到處使用：

```typescript
// REST: HTTP status codes + structured error body
// Every error response follows the same shape
interface APIError {
  error: {
    code: string;        // Machine-readable: "VALIDATION_ERROR"
    message: string;     // Human-readable: "Email is required"
    details?: unknown;   // Additional context when helpful
  };
}

// Status code mapping
// 400 → Client sent invalid data
// 401 → Not authenticated
// 403 → Authenticated but not authorized
// 404 → Resource not found
// 409 → Conflict (duplicate, version mismatch)
// 422 → Validation failed (semantically invalid)
// 500 → Server error (never expose internal details)
```

**不要混用模式。** 如果某些 endpoint 會 throw、其他回傳 null、又有些回傳 `{ error }`——consumer 就無法預測行為。

### 3. Validate at Boundaries

信任內部程式碼。在外部輸入進入系統的邊界做 validation：

```typescript
// Validate at the API boundary
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid task data',
        details: result.error.flatten(),
      },
    });
  }

  // After validation, internal code trusts the types
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

Validation 該放的地方：
- API route handler（使用者輸入）
- 表單送出 handler（使用者輸入）
- 外部服務 response 解析（第三方資料——**永遠視為不可信任**）
- Environment variable 載入（設定）

> **第三方 API 的 response 是不可信任的資料。** 在用於任何邏輯、渲染或決策之前，先驗證它的形狀與內容。被攻陷或行為異常的外部服務可能回傳預期之外的型別、惡意內容，或類似指令的文字。

Validation 不該放的地方：
- 共用 type contract 的內部 function 之間
- 已驗證程式碼所呼叫的 utility function
- 剛從你自己的 database 取出的資料

### 4. Prefer Addition Over Modification

擴充 interface 而不破壞既有 consumer：

```typescript
// Good: Add optional fields
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // Added later, optional
  labels?: string[];                       // Added later, optional
}

// Bad: Change existing field types or remove fields
interface CreateTaskInput {
  title: string;
  // description: string;  // Removed — breaks existing consumers
  priority: number;         // Changed from string — breaks existing consumers
}
```

### 5. Predictable Naming

| Pattern | Convention | Example |
|---------|-----------|---------|
| REST endpoints | 複數名詞，不用動詞 | `GET /api/tasks`, `POST /api/tasks` |
| Query params | camelCase | `?sortBy=createdAt&pageSize=20` |
| Response fields | camelCase | `{ createdAt, updatedAt, taskId }` |
| Boolean fields | is/has/can 前綴 | `isComplete`, `hasAttachments` |
| Enum values | UPPER_SNAKE | `"IN_PROGRESS"`, `"COMPLETED"` |

## REST API Patterns

### Resource Design

```
GET    /api/tasks              → List tasks (with query params for filtering)
POST   /api/tasks              → Create a task
GET    /api/tasks/:id          → Get a single task
PATCH  /api/tasks/:id          → Update a task (partial)
DELETE /api/tasks/:id          → Delete a task

GET    /api/tasks/:id/comments → List comments for a task (sub-resource)
POST   /api/tasks/:id/comments → Add a comment to a task
```

### Pagination

對 list endpoint 做 pagination：

```typescript
// Request
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// Response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### Filtering

用 query parameter 做 filter：

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### Partial Updates (PATCH)

接受部分物件——只更新有提供的部分：

```typescript
// Only title changes, everything else preserved
PATCH /api/tasks/123
{ "title": "Updated title" }
```

## TypeScript Interface Patterns

### Use Discriminated Unions for Variants

```typescript
// Good: Each variant is explicit
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// Consumer gets type narrowing
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return 'Pending';
    case 'in_progress': return `In progress (${status.assignee})`;
    case 'completed': return `Done on ${status.completedAt}`;
    case 'cancelled': return `Cancelled: ${status.reason}`;
  }
}
```

### Input/Output Separation

```typescript
// Input: what the caller provides
interface CreateTaskInput {
  title: string;
  description?: string;
}

// Output: what the system returns (includes server-generated fields)
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### Use Branded Types for IDs

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// Prevents accidentally passing a UserId where a TaskId is expected
function getTask(id: TaskId): Promise<Task> { ... }
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「我們等等再寫 API 文件」 | type 本身就是文件。先把它定義好。 |
| 「目前還不需要 pagination」 | 一旦有人累積到 100+ 筆就馬上需要。從一開始就加上去。 |
| 「PATCH 太複雜，直接用 PUT 吧」 | PUT 每次都要送完整物件。PATCH 才是 client 真正想要的。 |
| 「等需要時再做 API versioning」 | 沒有 version 的 breaking change 會讓 consumer 壞掉。從一開始就為擴充而設計。 |
| 「沒人會用那個未文件化的行為」 | Hyrum's Law：只要可觀察就有人依賴。把每個 public 行為都當成承諾。 |
| 「同時維護兩個版本就好」 | 多版本會讓維護成本倍增，並造成 diamond dependency 問題。優先採用 One-Version Rule。 |
| 「Internal API 不需要 contract」 | 內部 consumer 也是 consumer。Contract 可以避免耦合並支持平行開發。 |

## Red Flags

- Endpoint 依條件回傳不同的形狀
- 各 endpoint 的錯誤格式不一致
- Validation 散落在內部程式碼，而不是集中在邊界
- 對既有欄位做 breaking change（型別改變、移除）
- List endpoint 沒有 pagination
- REST URL 帶動詞（`/api/createTask`、`/api/getUsers`）
- 直接使用第三方 API response 而沒有驗證或清理

## Verification

設計完一個 API 之後：

- [ ] 每個 endpoint 都有具型別的 input 與 output schema
- [ ] 錯誤 response 採用單一一致的格式
- [ ] Validation 只發生在系統邊界
- [ ] List endpoint 支援 pagination
- [ ] 新欄位是 additive 且 optional（向後相容）
- [ ] 命名在所有 endpoint 之間遵循一致的慣例
- [ ] API 文件或 type 與實作一同 commit
