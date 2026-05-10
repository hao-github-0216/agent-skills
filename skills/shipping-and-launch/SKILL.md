---
name: shipping-and-launch
description: 進行 pre-launch readiness check。在這個 fork 的領域中，「shipping」主要指 submit SLURM job 至 OSU HPC，而非 production deploy。在啟動長時間 ML training run、escalating partition、或（少數情況）真的要 deploy production 軟體之前使用。English: Prepares pre-launch readiness checks. In this fork's domain "shipping" primarily means submitting a SLURM job to OSU HPC, not a production deploy. Use before kicking off a long ML training run, when escalating partition, or (rarely) when actually deploying production software.
---

# Shipping and Launch

## Overview

在這個 fork 的領域中，「ship」幾乎一律是指：**submit 一個非平凡的 SLURM job 到 OSU HPC。** 一個 Phase-1+ run 失敗的代價很高 — 排隊時間、A100/H100 時數、以及為了發現一個 3 分鐘 sanity check 就能抓到的 config bug 而等了好幾小時。目標與 production deploy 一樣：可逆、可觀察、可漸進，雖然底層是 SLURM 而不是 Kubernetes。

`/ship` slash command 的 parallel fan-out（code-reviewer + security-auditor + test-engineer）對於 code-quality-gate 仍有幫助；只是別期望那些 personas 知道 CUDA 正確性或 vault 整潔度 — 這些是本 skill 下方 checklist 補上的部分。

## HPC experiment run 的 pre-launch checklist（先用這個）

在 `bash ~/Bridge2HPC/hpc/submit.sh <project> <tag>` 之前過一次這份清單。任何一項未勾選就不要 submit。

### Code state
- [ ] 本機 repo commit 乾淨（`git status` 為空）
- [ ] 已 push 到 origin（`git push` 成功；HPC 端 `git pull` 才看得到）
- [ ] 沒有來自 sanity run 的殘留 `print()` / `breakpoint()` / debug-only flag
- [ ] Determinism：seeds 已設（torch + numpy + random + cuda），必要時設 `torch.use_deterministic_algorithms`

### Vault cell
- [ ] `~/vault/<Project>/Experiments/<tag>.md` 已存在（在 submit 前建立）
- [ ] 以白話文寫下 hypothesis（一句話）
- [ ] 已貼上 config dump（resolved values，不只是 CLI flag）
- [ ] 記錄 phase / partition（sanity / ampere / dgxh）並附上理由
- [ ] 記下 expected wall time — 這樣才能判斷 job 是否卡住

### Phase gates
- [ ] Phase 0 sanity 已在 howardserver 本機 GPU 上（3060 Ti，≤30 min）以「這份程式碼」通過
- [ ] 若要 escalate 到 `dgxh`：先前的 `ampere` run 實測 >6h，且 H100 上的 cuDNN smoke test 已通過。不可從 sanity 直接跳到 H100。
- [ ] partition 不是 `dgxh200`（禁用），且非 smoke run 不可使用 `preempt`

### Reproducibility
- [ ] code commit hash 已記錄在 vault cell
- [ ] 已存下 conda env 名稱 + `pip freeze` 快照（或 reference 一份 env file）
- [ ] data 版本 / 來源已釘住（不要使用 "latest" 的 data symlink）

### Failure visibility
- [ ] `~/<repo>/logs/<tag>.{out,err}` 路徑已存在或將被建立
- [ ] howardserver 上的 `poll.sh` cron 仍存活（否則結果不會自動回流）
- [ ] 至少在 train loop 內部每 N 步 `git push` 中間 metrics，使局部失敗可被觀察到

### Submit
- [ ] 透過 `~/Bridge2HPC/hpc/submit.sh` submit，而非直接 ssh
- [ ] tag 與 vault cell 檔名相符（互相對照）

### Post-submit (within 5 minutes)
- [ ] `squeue -u hunghao` 顯示 job 處於 R 或 PD 狀態
- [ ] `.out` log 前幾行顯示預期 env、GPU 偵測到、dataset 已載入
- [ ] 若第一步 loss 為 NaN 或卡住，立即 `scancel` — 不要燒掉排隊時間

## experiment 的 rollback plan

對 research run 來說，「rollback」= `scancel <jobid>` + 把 vault cell 狀態從 `running` 翻成 `failed` + 寫 1–2 句的 diagnosis。這是不可妥協的：一個半死不活、悄悄失敗的 run 會污染整個 ledger。

---

下方剩餘段落是上游 repo 的通用 production-launch checklist。只在你真的在 deploy production 軟體時套用（在這個 fork 領域很少見 — research repo 不會 ship 給使用者）。

## When to Use (production-deploy 變體)

- 第一次把功能 deploy 到 production
- 對使用者推出重要變更
- 遷移資料或 infrastructure
- 開放 beta 或 early access 計畫
- 任何具風險的 deployment（其實全部都是）

## The Pre-Launch Checklist

### Code Quality

- [ ] 所有測試通過（unit、integration、e2e）
- [ ] build 成功且無 warning
- [ ] lint 與 type check 通過
- [ ] code 已被 review 並核准
- [ ] launch 前該解決的 TODO 註解都已清除
- [ ] production code 中沒有 `console.log` debug 語句
- [ ] error handling 涵蓋預期的失敗模式

### Security

- [ ] 原始碼或版本控制中沒有 secret
- [ ] `npm audit` 沒有 critical 或 high 的漏洞
- [ ] 所有面向使用者的 endpoint 都有 input validation
- [ ] authentication 與 authorization 檢查到位
- [ ] security headers 已設定（CSP、HSTS 等）
- [ ] authentication endpoint 有 rate limiting
- [ ] CORS 設定限制在特定 origin（非 wildcard）

### Performance

- [ ] Core Web Vitals 落在「Good」門檻內
- [ ] 關鍵路徑沒有 N+1 query
- [ ] 圖片已優化（壓縮、響應式尺寸、lazy loading）
- [ ] bundle size 在預算內
- [ ] database query 有適當的 index
- [ ] 對靜態資源與重複 query 啟用 caching

### Accessibility

- [ ] 所有可互動元素都能用鍵盤操作
- [ ] screen reader 能傳達頁面內容與結構
- [ ] 顏色對比達到 WCAG 2.1 AA（文字 4.5:1）
- [ ] modal 與動態內容的 focus management 正確
- [ ] error message 具描述性並與表單欄位連結
- [ ] axe-core 或 Lighthouse 沒有 accessibility 警告

### Infrastructure

- [ ] production 環境的環境變數已設
- [ ] database migration 已 apply（或就緒可 apply）
- [ ] DNS 與 SSL 已設定
- [ ] 靜態資源的 CDN 已設定
- [ ] logging 與 error reporting 已設定
- [ ] health check endpoint 存在且可回應

### Documentation

- [ ] README 已更新任何新的 setup 需求
- [ ] API documentation 為最新
- [ ] 任何架構決策都有 ADR
- [ ] changelog 已更新
- [ ] 對使用者公開的文件已更新（若適用）

## Feature Flag Strategy

以 feature flag 出貨，把 deployment 與 release 解耦：

```typescript
// Feature flag check
const flags = await getFeatureFlags(userId);

if (flags.taskSharing) {
  // New feature: task sharing
  return <TaskSharingPanel task={task} />;
}

// Default: existing behavior
return null;
```

**Feature flag 生命週期：**

```
1. DEPLOY with flag OFF     → Code is in production but inactive
2. ENABLE for team/beta     → Internal testing in production environment
3. GRADUAL ROLLOUT          → 5% → 25% → 50% → 100% of users
4. MONITOR at each stage    → Watch error rates, performance, user feedback
5. CLEAN UP                 → Remove flag and dead code path after full rollout
```

**規則：**
- 每個 feature flag 都有 owner 與到期日
- 全量 rollout 之後 2 週內把 flag 清掉
- 不要 nesting feature flag（會產生指數級組合）
- 在 CI 中測試 flag 的兩種狀態（on 與 off）

## Staged Rollout

### The Rollout Sequence

```
1. DEPLOY to staging
   └── Full test suite in staging environment
   └── Manual smoke test of critical flows

2. DEPLOY to production (feature flag OFF)
   └── Verify deployment succeeded (health check)
   └── Check error monitoring (no new errors)

3. ENABLE for team (flag ON for internal users)
   └── Team uses the feature in production
   └── 24-hour monitoring window

4. CANARY rollout (flag ON for 5% of users)
   └── Monitor error rates, latency, user behavior
   └── Compare metrics: canary vs. baseline
   └── 24-48 hour monitoring window
   └── Advance only if all thresholds pass (see table below)

5. GRADUAL increase (25% -> 50% -> 100%)
   └── Same monitoring at each step
   └── Ability to roll back to previous percentage at any point

6. FULL rollout (flag ON for all users)
   └── Monitor for 1 week
   └── Clean up feature flag
```

### Rollout Decision Thresholds

每個階段使用以下門檻決定是 advance、hold、還是 rollback：

| Metric | Advance (green) | Hold and investigate (yellow) | Roll back (red) |
|--------|-----------------|-------------------------------|-----------------|
| Error rate | 與 baseline 差距在 10% 以內 | 高於 baseline 10-100% | 超過 baseline 2 倍 |
| P95 latency | 與 baseline 差距在 20% 以內 | 高於 baseline 20-50% | 高於 baseline 50% 以上 |
| Client JS errors | 沒有新型 error | 新 error 出現於 <0.1% session | 新 error 出現於 >0.1% session |
| Business metrics | 持平或正向 | 下滑 <5%（可能是雜訊） | 下滑 >5% |

### When to Roll Back

下列情況要立即 roll back：
- error rate 超過 baseline 2 倍
- P95 latency 增加超過 50%
- 使用者回報問題暴增
- 偵測到資料完整性問題
- 發現 security 漏洞

## Monitoring and Observability

### What to Monitor

```
Application metrics:
├── Error rate (total and by endpoint)
├── Response time (p50, p95, p99)
├── Request volume
├── Active users
└── Key business metrics (conversion, engagement)

Infrastructure metrics:
├── CPU and memory utilization
├── Database connection pool usage
├── Disk space
├── Network latency
└── Queue depth (if applicable)

Client metrics:
├── Core Web Vitals (LCP, INP, CLS)
├── JavaScript errors
├── API error rates from client perspective
└── Page load time
```

### Error Reporting

```typescript
// Set up error boundary with reporting
class ErrorBoundary extends React.Component {
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Report to error tracking service
    reportError(error, {
      componentStack: info.componentStack,
      userId: getCurrentUser()?.id,
      page: window.location.pathname,
    });
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback onRetry={() => this.setState({ hasError: false })} />;
    }
    return this.props.children;
  }
}

// Server-side error reporting
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  reportError(err, {
    method: req.method,
    url: req.url,
    userId: req.user?.id,
  });

  // Don't expose internals to users
  res.status(500).json({
    error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' },
  });
});
```

### Post-Launch Verification

launch 後第一小時內：

```
1. Check health endpoint returns 200
2. Check error monitoring dashboard (no new error types)
3. Check latency dashboard (no regression)
4. Test the critical user flow manually
5. Verify logs are flowing and readable
6. Confirm rollback mechanism works (dry run if possible)
```

## Rollback Strategy

每次 deploy 都應該在事前準備好 rollback plan：

```markdown
## Rollback Plan for [Feature/Release]

### Trigger Conditions
- Error rate > 2x baseline
- P95 latency > [X]ms
- User reports of [specific issue]

### Rollback Steps
1. Disable feature flag (if applicable)
   OR
1. Deploy previous version: `git revert <commit> && git push`
2. Verify rollback: health check, error monitoring
3. Communicate: notify team of rollback

### Database Considerations
- Migration [X] has a rollback: `npx prisma migrate rollback`
- Data inserted by new feature: [preserved / cleaned up]

### Time to Rollback
- Feature flag: < 1 minute
- Redeploy previous version: < 5 minutes
- Database rollback: < 15 minutes
```
## See Also

- security pre-launch 檢查請見 `references/security-checklist.md`
- performance pre-launch checklist 請見 `references/performance-checklist.md`
- launch 前的 accessibility 驗證請見 `references/accessibility-checklist.md`

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「在 staging 跑得起來，production 一定也行」 | production 的資料、流量模式、edge case 都不一樣。deploy 後要持續 monitor。 |
| 「這個不需要 feature flag」 | 任何功能都受惠於 kill switch。連「簡單」的變更也可能搞壞東西。 |
| 「monitoring 是額外負擔」 | 沒有 monitoring，你只能從使用者抱怨中發現問題，而非從 dashboard。 |
| 「之後再加 monitoring」 | launch 前就要加上。看不到的東西無法 debug。 |
| 「rollback 等於承認失敗」 | rollback 是負責任的工程。Ship 一個壞掉的功能才是失敗。 |

## Red Flags

- 沒有 rollback plan 就 deploy
- production 上沒有 monitoring 或 error reporting
- big-bang release（一次全部上線、沒有 staging）
- 沒設到期日或 owner 的 feature flag
- 沒人在 deploy 後第一小時 monitor
- production 環境設定靠記憶完成而不是用 code
- 「現在禮拜五下午，順手 ship 一下」

## Verification

deploy 之前：

- [ ] pre-launch checklist 已完成（每個段落都過關）
- [ ] feature flag 已設定（若適用）
- [ ] rollback plan 已記錄
- [ ] monitoring dashboard 已就緒
- [ ] 團隊已被通知此次 deployment

deploy 之後：

- [ ] health check 回傳 200
- [ ] error rate 正常
- [ ] latency 正常
- [ ] 關鍵 user flow 可運作
- [ ] log 持續流入
- [ ] rollback 已測試或確認就緒
