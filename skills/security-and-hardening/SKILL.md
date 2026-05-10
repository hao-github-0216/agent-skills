---
name: security-and-hardening
description: 強化程式碼以抵禦漏洞。處理 user input、authentication、資料儲存或外部整合時使用。在打造任何接受不可信資料、管理 user session 或與第三方服務互動的功能時也適用。English: Hardens code against vulnerabilities. Use when handling user input, authentication, data storage, or external integrations. Use when building any feature that accepts untrusted data, manages user sessions, or interacts with third-party services.
---

# Security and Hardening

## Overview

以 security 為優先的 web application 開發實務。視所有外部輸入為敵意、視所有 secret 為神聖、視所有 authorization check 為必要。Security 不是某個階段 — 它是每一行碰到 user data、authentication 或外部系統的程式碼都要遵守的限制。

## When to Use

- 打造任何接受 user input 的東西
- 實作 authentication 或 authorization
- 儲存或傳輸敏感資料
- 與外部 API 或服務整合
- 加入檔案上傳、webhook 或 callback
- 處理付款或 PII 資料

## The Three-Tier Boundary System

### Always Do (No Exceptions)

- **驗證所有外部輸入**，在系統邊界進行（API routes、form handlers）
- **所有 database query 都要 parameterize** — 永遠不要把 user input 串接進 SQL
- **對輸出做 encoding** 以防 XSS（用 framework 的自動跳脫，不要繞過它）
- 所有外部通訊都使用 **HTTPS**
- **以 bcrypt/scrypt/argon2 hash 密碼**（永不儲存明文）
- **設定 security headers**（CSP、HSTS、X-Frame-Options、X-Content-Type-Options）
- **session 使用 httpOnly、secure、sameSite cookie**
- 每次發版前執行 **`npm audit`**（或同等工具）

### Ask First (Requires Human Approval)

- 加入新的 authentication flow 或變更 auth 邏輯
- 儲存新類別的敏感資料（PII、付款資訊）
- 加入新的外部服務整合
- 變更 CORS 設定
- 加入檔案上傳處理
- 修改 rate limiting 或 throttling
- 授予提升的權限或角色

### Never Do

- **絕不把 secret commit** 進版本控制（API key、密碼、token）
- **絕不 log 敏感資料**（密碼、token、完整信用卡號）
- **絕不把 client-side validation 當作 security 邊界**
- **絕不為了方便而停用 security headers**
- **絕不對 user-provided data 使用 `eval()` 或 `innerHTML`**
- **絕不把 session 存在 client 可存取的儲存空間**（如把 auth token 放在 localStorage）
- **絕不向使用者暴露 stack trace** 或內部錯誤細節

## OWASP Top 10 Prevention

### 1. Injection (SQL, NoSQL, OS Command)

```typescript
// BAD: SQL injection via string concatenation
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// GOOD: Parameterized query
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// GOOD: ORM with parameterized input
const user = await prisma.user.findUnique({ where: { id: userId } });
```

### 2. Broken Authentication

```typescript
// Password hashing
import { hash, compare } from 'bcrypt';

const SALT_ROUNDS = 12;
const hashedPassword = await hash(plaintext, SALT_ROUNDS);
const isValid = await compare(plaintext, hashedPassword);

// Session management
app.use(session({
  secret: process.env.SESSION_SECRET,  // From environment, not code
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,     // Not accessible via JavaScript
    secure: true,       // HTTPS only
    sameSite: 'lax',    // CSRF protection
    maxAge: 24 * 60 * 60 * 1000,  // 24 hours
  },
}));
```

### 3. Cross-Site Scripting (XSS)

```typescript
// BAD: Rendering user input as HTML
element.innerHTML = userInput;

// GOOD: Use framework auto-escaping (React does this by default)
return <div>{userInput}</div>;

// If you MUST render HTML, sanitize first
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
```

### 4. Broken Access Control

```typescript
// Always check authorization, not just authentication
app.patch('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await taskService.findById(req.params.id);

  // Check that the authenticated user owns this resource
  if (task.ownerId !== req.user.id) {
    return res.status(403).json({
      error: { code: 'FORBIDDEN', message: 'Not authorized to modify this task' }
    });
  }

  // Proceed with update
  const updated = await taskService.update(req.params.id, req.body);
  return res.json(updated);
});
```

### 5. Security Misconfiguration

```typescript
// Security headers (use helmet for Express)
import helmet from 'helmet';
app.use(helmet());

// Content Security Policy
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],  // Tighten if possible
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'"],
  },
}));

// CORS — restrict to known origins
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));
```

### 6. Sensitive Data Exposure

```typescript
// Never return sensitive fields in API responses
function sanitizeUser(user: UserRecord): PublicUser {
  const { passwordHash, resetToken, ...publicFields } = user;
  return publicFields;
}

// Use environment variables for secrets
const API_KEY = process.env.STRIPE_API_KEY;
if (!API_KEY) throw new Error('STRIPE_API_KEY not configured');
```

## Input Validation Patterns

### 在邊界做 Schema Validation

```typescript
import { z } from 'zod';

const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  description: z.string().max(2000).optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().datetime().optional(),
});

// Validate at the route handler
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid input',
        details: result.error.flatten(),
      },
    });
  }
  // result.data is now typed and validated
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

### 檔案上傳安全

```typescript
// Restrict file types and sizes
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateUpload(file: UploadedFile) {
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new ValidationError('File type not allowed');
  }
  if (file.size > MAX_SIZE) {
    throw new ValidationError('File too large (max 5MB)');
  }
  // Don't trust the file extension — check magic bytes if critical
}
```

## Triaging npm audit Results

並非所有 audit 結果都需要立刻處理。可使用以下決策樹：

```
npm audit reports a vulnerability
├── Severity: critical or high
│   ├── Is the vulnerable code reachable in your app?
│   │   ├── YES --> Fix immediately (update, patch, or replace the dependency)
│   │   └── NO (dev-only dep, unused code path) --> Fix soon, but not a blocker
│   └── Is a fix available?
│       ├── YES --> Update to the patched version
│       └── NO --> Check for workarounds, consider replacing the dependency, or add to allowlist with a review date
├── Severity: moderate
│   ├── Reachable in production? --> Fix in the next release cycle
│   └── Dev-only? --> Fix when convenient, track in backlog
└── Severity: low
    └── Track and fix during regular dependency updates
```

**關鍵問題：**
- 該漏洞函式是否真的會在你的程式碼路徑中被呼叫？
- 該 dependency 是 runtime dependency 還是僅 dev-only？
- 在你部署的情境下該漏洞是否可被利用（例如某個 server-side 漏洞用在 client-only app 上）？

當你決定延後修復時，記下原因並設定一個 review 日期。

## Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

// General API rate limit
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                   // 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
}));

// Stricter limit for auth endpoints
app.use('/api/auth/', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 10 attempts per 15 minutes
}));
```

## Secrets Management

```
.env files:
  ├── .env.example  → Committed (template with placeholder values)
  ├── .env          → NOT committed (contains real secrets)
  └── .env.local    → NOT committed (local overrides)

.gitignore must include:
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
```

**commit 前永遠先檢查：**
```bash
# Check for accidentally staged secrets
git diff --cached | grep -i "password\|secret\|api_key\|token"
```

## Security Review Checklist

```markdown
### Authentication
- [ ] Passwords hashed with bcrypt/scrypt/argon2 (salt rounds ≥ 12)
- [ ] Session tokens are httpOnly, secure, sameSite
- [ ] Login has rate limiting
- [ ] Password reset tokens expire

### Authorization
- [ ] Every endpoint checks user permissions
- [ ] Users can only access their own resources
- [ ] Admin actions require admin role verification

### Input
- [ ] All user input validated at the boundary
- [ ] SQL queries are parameterized
- [ ] HTML output is encoded/escaped

### Data
- [ ] No secrets in code or version control
- [ ] Sensitive fields excluded from API responses
- [ ] PII encrypted at rest (if applicable)

### Infrastructure
- [ ] Security headers configured (CSP, HSTS, etc.)
- [ ] CORS restricted to known origins
- [ ] Dependencies audited for vulnerabilities
- [ ] Error messages don't expose internals
```
## See Also

關於詳盡的 security checklist 與 pre-commit 驗證步驟，請見 `references/security-checklist.md`。

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「這只是內部工具，security 沒那麼重要」 | 內部工具一樣會被入侵。攻擊者鎖定最弱的一環。 |
| 「之後再補 security」 | 事後補強的成本是當下做好的 10 倍。現在就加。 |
| 「沒人會試圖攻擊這個」 | 自動化掃描器會找到它。靠隱蔽性的 security 不算 security。 |
| 「framework 會處理 security」 | framework 提供工具，而不是保證。你還是得正確使用它們。 |
| 「這只是 prototype」 | Prototype 會變 production。從第一天就建立 security 習慣。 |

## Red Flags

- user input 直接傳進 database query、shell command 或 HTML render
- secret 出現在原始碼或 commit 紀錄裡
- API endpoint 沒有 authentication 或 authorization 檢查
- 缺少 CORS 設定，或使用 wildcard（`*`）origin
- authentication endpoint 沒有 rate limiting
- 將 stack trace 或內部錯誤暴露給使用者
- 使用了已知含 critical 漏洞的 dependency

## Verification

實作完 security 相關程式碼後：

- [ ] `npm audit` 沒有 critical 或 high 的漏洞
- [ ] 原始碼或 git 歷史中沒有 secret
- [ ] 所有 user input 都在系統邊界經過驗證
- [ ] 所有受保護 endpoint 都檢查 authentication 與 authorization
- [ ] response 中存在 security headers（用 browser DevTools 檢查）
- [ ] error response 不會暴露內部細節
- [ ] auth endpoint 啟用 rate limiting
