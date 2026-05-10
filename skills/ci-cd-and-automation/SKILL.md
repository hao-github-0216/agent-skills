---
name: ci-cd-and-automation
description: 自動化 CI/CD pipeline 設置。用於建立或修改 build 與 deployment pipeline 時。用於需要自動化品質門檻、在 CI 設定 test runner，或建立 deployment 策略時。English: Automates CI/CD pipeline setup. Use when setting up or modifying build and deployment pipelines. Use when you need to automate quality gates, configure test runners in CI, or establish deployment strategies.
---

# CI/CD and Automation

## Overview

把品質門檻自動化，讓每個變更都必須通過測試、lint、type check 與 build 才能進到 production。CI/CD 是其他所有 skill 的執行機制——它能補捉人類與 agent 漏掉的東西，而且在每一次變更上都一致地做這件事。

**Shift Left：** 盡早在 pipeline 中抓出問題。在 lint 階段抓到的 bug 只花幾分鐘；同樣的 bug 在 production 抓到要花好幾小時。把檢查往上游推——靜態分析在測試之前、測試在 staging 之前、staging 在 production 之前。

**越快越安全：** 較小的批次與更頻繁的 release 會降低風險而非升高。一次 deploy 含 3 個變更比一次含 30 個更容易 debug。頻繁 release 能對 release 流程本身建立信心。

## When to Use

- 為新專案建立 CI pipeline
- 新增或修改自動化檢查
- 設定 deployment pipeline
- 當變更應該觸發自動驗證時
- Debug CI 失敗

## The Quality Gate Pipeline

每個變更在 merge 之前都會經過這些 gate：

```
Pull Request Opened
    │
    ▼
┌─────────────────┐
│   LINT CHECK     │  eslint, prettier
│   ↓ pass         │
│   TYPE CHECK     │  tsc --noEmit
│   ↓ pass         │
│   UNIT TESTS     │  jest/vitest
│   ↓ pass         │
│   BUILD          │  npm run build
│   ↓ pass         │
│   INTEGRATION    │  API/DB tests
│   ↓ pass         │
│   E2E (optional) │  Playwright/Cypress
│   ↓ pass         │
│   SECURITY AUDIT │  npm audit
│   ↓ pass         │
│   BUNDLE SIZE    │  bundlesize check
└─────────────────┘
    │
    ▼
  Ready for review
```

**沒有任何 gate 可以略過。** Lint 失敗就修 lint——不要關掉那條規則。Test 失敗就修程式碼——不要 skip 那個 test。

## GitHub Actions Configuration

### Basic CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Security audit
        run: npm audit --audit-level=high
```

### With Database Integration Tests

```yaml
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: ci_user
          POSTGRES_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
```

> **注意：** 即使是 CI 專用的 test database，也要用 GitHub Secrets 來存放憑證，而不是寫死在程式碼裡。這能養成好習慣，並避免不小心在其他情境重用測試憑證。

### E2E Tests

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      - name: Build
        run: npm run build
      - name: Run E2E tests
        run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

## Feeding CI Failures Back to Agents

CI 搭配 AI agent 的威力在於 feedback loop。當 CI 失敗時：

```
CI fails
    │
    ▼
Copy the failure output
    │
    ▼
Feed it to the agent:
"The CI pipeline failed with this error:
[paste specific error]
Fix the issue and verify locally before pushing again."
    │
    ▼
Agent fixes → pushes → CI runs again
```

**主要 pattern：**

```
Lint failure → Agent runs `npm run lint --fix` and commits
Type error  → Agent reads the error location and fixes the type
Test failure → Agent follows debugging-and-error-recovery skill
Build error → Agent checks config and dependencies
```

## Deployment Strategies

### Preview Deployments

每個 PR 都產生一份 preview deployment 供手動測試：

```yaml
# Deploy preview on PR (Vercel/Netlify/etc.)
deploy-preview:
  runs-on: ubuntu-latest
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - name: Deploy preview
      run: npx vercel --token=${{ secrets.VERCEL_TOKEN }}
```

### Feature Flags

Feature flag 把 deploy 與 release 解耦。將未完成或有風險的功能藏在 flag 後面 deploy，這樣你可以：

- **Ship 程式碼但不啟用它。** 早早 merge 到 main，準備好再啟用。
- **不需要重新 deploy 就能 rollback。** 關掉 flag 就好，不用 revert 程式碼。
- **對新功能做 canary。** 先開放給 1% 使用者、然後 10%、再 100%。
- **執行 A/B test。** 比較有無此功能的行為差異。

```typescript
// Simple feature flag pattern
if (featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**Flag 的生命週期：** 建立 → 開給內部測試 → canary → 全面 rollout → 移除 flag 與已死的程式碼。永遠不死的 flag 會變成技術債——建立的同時就設一個清理日期。

### Staged Rollouts

```
PR merged to main
    │
    ▼
  Staging deployment (auto)
    │ Manual verification
    ▼
  Production deployment (manual trigger or auto after staging)
    │
    ▼
  Monitor for errors (15-minute window)
    │
    ├── Errors detected → Rollback
    └── Clean → Done
```

### Rollback Plan

每次 deployment 都應該可回退：

```yaml
# Manual rollback workflow
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback deployment
        run: |
          # Deploy the specified previous version
          npx vercel rollback ${{ inputs.version }}
```

## Environment Management

```
.env.example       → Committed (template for developers)
.env                → NOT committed (local development)
.env.test           → Committed (test environment, no real secrets)
CI secrets          → Stored in GitHub Secrets / vault
Production secrets  → Stored in deployment platform / vault
```

CI 絕不應該擁有 production 的 secret。CI 測試請使用獨立的 secret。

## Automation Beyond CI

### Dependabot / Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

### Build Cop Role

指定一個人負責讓 CI 保持綠燈。當 build 壞掉時，Build Cop 的工作是修好或 revert——不是讓造成壞掉的那個人負責。這能防止壞掉的 build 累積，因為大家都以為「會有別人去修」。

### PR Checks

- **Required reviews：** Merge 前至少一個人 approve
- **Required status checks：** Merge 前 CI 必須通過
- **Branch protection：** 不允許 force-push 到 main
- **Auto-merge：** 通過所有檢查並 approve 後自動 merge

## CI Optimization

當 pipeline 超過 10 分鐘時，依影響大小依序套用以下策略：

```
Slow CI pipeline?
├── Cache dependencies
│   └── Use actions/cache or setup-node cache option for node_modules
├── Run jobs in parallel
│   └── Split lint, typecheck, test, build into separate parallel jobs
├── Only run what changed
│   └── Use path filters to skip unrelated jobs (e.g., skip e2e for docs-only PRs)
├── Use matrix builds
│   └── Shard test suites across multiple runners
├── Optimize the test suite
│   └── Remove slow tests from the critical path, run them on a schedule instead
└── Use larger runners
    └── GitHub-hosted larger runners or self-hosted for CPU-heavy builds
```

**範例：caching 與 parallelism**
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --coverage
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「CI 太慢了」 | 把 pipeline 優化（見下方 CI Optimization），不要 skip。5 分鐘的 pipeline 能省掉好幾小時的 debug。 |
| 「這個變更很小，跳過 CI 吧」 | 小變更也會弄壞 build。而且小變更跑 CI 本來就快。 |
| 「這個 test 不穩，重跑就好」 | Flaky test 會掩蓋真正的 bug 並浪費所有人時間。修好它的不穩定。 |
| 「我們之後再加 CI」 | 沒有 CI 的專案會累積壞掉的狀態。第一天就建好。 |
| 「手動測試就夠了」 | 手動測試無法擴展也不可重複。能自動化的就自動化。 |

## Red Flags

- 專案沒有 CI pipeline
- CI 失敗被忽略或被靜音
- 為了讓 pipeline 過而把測試 disable
- 沒經過 staging 驗證就直接 deploy 到 production
- 沒有 rollback 機制
- Secret 存在程式碼或 CI 設定檔（而非 secrets manager）
- CI 跑很久卻沒人去優化

## Verification

建立或修改 CI 之後：

- [ ] 所有品質門檻都到位（lint、type、test、build、audit）
- [ ] Pipeline 在每個 PR 與每次推到 main 時執行
- [ ] 失敗會 block merge（branch protection 已設定）
- [ ] CI 結果會回饋進開發迴圈
- [ ] Secret 存在 secrets manager 而非程式碼裡
- [ ] Deployment 有 rollback 機制
- [ ] 測試套件的 pipeline 在 10 分鐘內跑完
