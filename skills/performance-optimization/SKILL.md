---
name: performance-optimization
description: 為應用程式做 performance 最佳化。當有 performance 需求、懷疑有 performance regression、或需要改善 Core Web Vitals 或 load time 時使用。當 profiling 找出需要修的 bottleneck 時使用。English: Optimizes application performance. Use when performance requirements exist, when you suspect performance regressions, or when Core Web Vitals or load times need improvement. Use when profiling reveals bottlenecks that need fixing.
---

# Performance Optimization

## Overview

最佳化前先量測。沒有量測的 performance 工作就是猜測 — 而猜測會導致 premature optimization，加上複雜度卻沒改善真正重要的東西。先 profile、找出真正的 bottleneck、修它、再量一次。只最佳化量測證明重要的東西。

## When to Use

- spec 中有 performance 需求（load time 預算、回應時間 SLA）
- 使用者或監控系統回報變慢
- Core Web Vitals 分數低於門檻
- 你懷疑某個變更引入了 regression
- 在打造處理大資料集或高流量的 feature

**何時不要用：** 在還沒有問題證據時不要最佳化。premature optimization 增加的複雜度成本，會大過它換來的 performance。

## Core Web Vitals Targets

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## The Optimization Workflow

```
1. MEASURE  → Establish baseline with real data
2. IDENTIFY → Find the actual bottleneck (not assumed)
3. FIX      → Address the specific bottleneck
4. VERIFY   → Measure again, confirm improvement
5. GUARD    → Add monitoring or tests to prevent regression
```

### Step 1: Measure

兩種互補的方法 — 都要用：

- **Synthetic（Lighthouse、DevTools Performance tab）：** 受控條件、可重現。最適合 CI 上的 regression 偵測與隔離特定問題。
- **RUM（web-vitals library、CrUX）：** 真實使用者在真實條件下的資料。是驗證修補真的改善了使用者體驗的必要條件。

**Frontend：**
```bash
# Synthetic: Lighthouse in Chrome DevTools (or CI)
# Chrome DevTools → Performance tab → Record
# Chrome DevTools MCP → Performance trace

# RUM: Web Vitals library in code
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

**Backend：**
```bash
# Response time logging
# Application Performance Monitoring (APM)
# Database query logging with timing

# Simple timing
console.time('db-query');
const result = await db.query(...);
console.timeEnd('db-query');
```

### Where to Start Measuring

用症狀來決定先量什麼：

```
What is slow?
├── First page load
│   ├── Large bundle? --> Measure bundle size, check code splitting
│   ├── Slow server response? --> Measure TTFB in DevTools Network waterfall
│   │   ├── DNS long? --> Add dns-prefetch / preconnect for known origins
│   │   ├── TCP/TLS long? --> Enable HTTP/2, check edge deployment, keep-alive
│   │   └── Waiting (server) long? --> Profile backend, check queries and caching
│   └── Render-blocking resources? --> Check network waterfall for CSS/JS blocking
├── Interaction feels sluggish
│   ├── UI freezes on click? --> Profile main thread, look for long tasks (>50ms)
│   ├── Form input lag? --> Check re-renders, controlled component overhead
│   └── Animation jank? --> Check layout thrashing, forced reflows
├── Page after navigation
│   ├── Data loading? --> Measure API response times, check for waterfalls
│   └── Client rendering? --> Profile component render time, check for N+1 fetches
└── Backend / API
    ├── Single endpoint slow? --> Profile database queries, check indexes
    ├── All endpoints slow? --> Check connection pool, memory, CPU
    └── Intermittent slowness? --> Check for lock contention, GC pauses, external deps
```

### Step 2: Identify the Bottleneck

依類別常見的 bottleneck：

**Frontend：**

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| LCP 慢 | 大圖、render-blocking 資源、server 慢 | 檢查 network waterfall、圖片大小 |
| CLS 高 | 沒寫尺寸的圖片、晚載入的內容、字型 shift | 檢查 layout shift attribution |
| INP 差 | main thread 上的重 JavaScript、大量 DOM 更新 | 在 Performance trace 裡找 long task |
| 初始載入慢 | bundle 大、network request 多 | 檢查 bundle size、code splitting |

**Backend：**

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| API 回應慢 | N+1 query、缺 index、未最佳化的 query | 檢查 database query log |
| 記憶體成長 | reference 洩漏、無上限 cache、payload 過大 | heap snapshot 分析 |
| CPU 飆高 | 同步的重運算、regex backtracking | CPU profiling |
| latency 高 | 缺 cache、重複運算、network hop | 把 request 跨 stack 做 trace |

### Step 3: Fix Common Anti-Patterns

#### N+1 Queries (Backend)

```typescript
// BAD: N+1 — one query per task for the owner
const tasks = await db.tasks.findMany();
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.ownerId } });
}

// GOOD: Single query with join/include
const tasks = await db.tasks.findMany({
  include: { owner: true },
});
```

#### Unbounded Data Fetching

```typescript
// BAD: Fetching all records
const allTasks = await db.tasks.findMany();

// GOOD: Paginated with limits
const tasks = await db.tasks.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: { createdAt: 'desc' },
});
```

#### Missing Image Optimization (Frontend)

```html
<!-- BAD: No dimensions, no format optimization -->
<img src="/hero.jpg" />

<!-- GOOD: Hero / LCP image — art direction + resolution switching, high priority -->
<!--
  Two techniques combined:
  - Art direction (media): different crop/composition per breakpoint
  - Resolution switching (srcset + sizes): right file size per screen density
-->
<picture>
  <!-- Mobile: portrait crop (8:10) -->
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.avif 400w, /hero-mobile-800.avif 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/avif"
  />
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.webp 400w, /hero-mobile-800.webp 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/webp"
  />
  <!-- Desktop: landscape crop (2:1) -->
  <source
    srcset="/hero-800.avif 800w, /hero-1200.avif 1200w, /hero-1600.avif 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/avif"
  />
  <source
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w, /hero-1600.webp 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/webp"
  />
  <img
    src="/hero-desktop.jpg"
    width="1200"
    height="600"
    fetchpriority="high"
    alt="Hero image description"
  />
</picture>

<!-- GOOD: Below-the-fold image — lazy loaded + async decoding -->
<img
  src="/content.webp"
  width="800"
  height="400"
  loading="lazy"
  decoding="async"
  alt="Content image description"
/>
```

#### Unnecessary Re-renders (React)

```tsx
// BAD: Creates new object on every render, causing children to re-render
function TaskList() {
  return <TaskFilters options={{ sortBy: 'date', order: 'desc' }} />;
}

// GOOD: Stable reference
const DEFAULT_OPTIONS = { sortBy: 'date', order: 'desc' } as const;
function TaskList() {
  return <TaskFilters options={DEFAULT_OPTIONS} />;
}

// Use React.memo for expensive components
const TaskItem = React.memo(function TaskItem({ task }: Props) {
  return <div>{/* expensive render */}</div>;
});

// Use useMemo for expensive computations
function TaskStats({ tasks }: Props) {
  const stats = useMemo(() => calculateStats(tasks), [tasks]);
  return <div>{stats.completed} / {stats.total}</div>;
}
```

#### Large Bundle Size

```typescript
// Modern bundlers (Vite, webpack 5+) handle named imports with tree-shaking automatically,
// provided the dependency ships ESM and is marked `sideEffects: false` in package.json.
// Profile before changing import styles — the real gains come from splitting and lazy loading.

// GOOD: Dynamic import for heavy, rarely-used features
const ChartLibrary = lazy(() => import('./ChartLibrary'));

// GOOD: Route-level code splitting wrapped in Suspense
const SettingsPage = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <SettingsPage />
    </Suspense>
  );
}
```

#### Missing Caching (Backend)

```typescript
// Cache frequently-read, rarely-changed data
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes
let cachedConfig: AppConfig | null = null;
let cacheExpiry = 0;

async function getAppConfig(): Promise<AppConfig> {
  if (cachedConfig && Date.now() < cacheExpiry) {
    return cachedConfig;
  }
  cachedConfig = await db.config.findFirst();
  cacheExpiry = Date.now() + CACHE_TTL;
  return cachedConfig;
}

// HTTP caching headers for static assets
app.use('/static', express.static('public', {
  maxAge: '1y',           // Cache for 1 year
  immutable: true,        // Never revalidate (use content hashing in filenames)
}));

// Cache-Control for API responses
res.set('Cache-Control', 'public, max-age=300'); // 5 minutes
```

## Performance Budget

訂下預算並強制執行：

```
JavaScript bundle: < 200KB gzipped (initial load)
CSS: < 50KB gzipped
Images: < 200KB per image (above the fold)
Fonts: < 100KB total
API response time: < 200ms (p95)
Time to Interactive: < 3.5s on 4G
Lighthouse Performance score: ≥ 90
```

**在 CI 上強制執行：**
```bash
# Bundle size check
npx bundlesize --config bundlesize.config.json

# Lighthouse CI
npx lhci autorun
```

## See Also

關於詳細的 performance checklist、最佳化指令、anti-pattern 參考，見 `references/performance-checklist.md`。


## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「我們之後再最佳化」 | performance 債會疊加。明顯的 anti-pattern 現在就修，micro-optimization 才延後。 |
| 「在我電腦上很快」 | 你的電腦不是使用者的。在有代表性的硬體與網路上 profile。 |
| 「這個最佳化很明顯」 | 如果你沒量，你就不知道。先 profile。 |
| 「使用者不會注意到 100ms」 | 研究顯示 100ms 的延遲會影響轉換率。使用者比你想的更有感。 |
| 「framework 會處理 performance」 | framework 能擋掉一些問題，但修不了 N+1 query 或過大的 bundle。 |

## Red Flags

- 沒有 profiling 資料佐證就最佳化
- 資料抓取裡有 N+1 query pattern
- 列表 endpoint 沒有 pagination
- 圖片沒有尺寸、lazy loading、或 responsive size
- bundle size 沒人 review 就一直成長
- production 上沒有 performance 監控
- `React.memo` 跟 `useMemo` 滿天飛（過度使用跟用得不夠一樣糟）

## Verification

任何 performance 相關變更之後：

- [ ] 有 before 跟 after 的量測（具體數字）
- [ ] 確切的 bottleneck 已被找出並處理
- [ ] Core Web Vitals 落在「Good」門檻內
- [ ] bundle size 沒有顯著成長
- [ ] 新的資料抓取程式碼裡沒有 N+1 query
- [ ] performance 預算在 CI 上通過（如有設定）
- [ ] 既有 test 仍然通過（最佳化沒破壞行為）
