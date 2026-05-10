---
name: deprecation-and-migration
description: 管理 deprecation 與 migration。在移除舊系統、API、或功能時使用。在把使用者從一個實作搬到另一個實作時使用。在決定要繼續維護還是 sunset 既有 code 時使用。English: Manages deprecation and migration. Use when removing old systems, APIs, or features. Use when migrating users from one implementation to another. Use when deciding whether to maintain or sunset existing code.
---

# Deprecation and Migration

## 概覽

Code 是 liability，不是 asset。每一行 code 都帶有持續維護成本 —— 要修 bug、更新 dependency、套 security patch、訓練新進工程師。Deprecation 就是把不再值得養的 code 拿掉的紀律，而 migration 則是把使用者安全地從舊搬到新的過程。

大多數工程組織擅長造東西，少數組織擅長拿掉東西。這個 skill 處理那個落差。

## 何時使用

- 用新系統、新 API、或新 library 取代舊的
- Sunset 已不再需要的功能
- 合併重複的實作
- 移除沒人擁有但所有人都依賴的 dead code
- 規劃新系統的生命週期（deprecation 規劃應在設計時就開始）
- 決定要繼續維護 legacy 系統還是投入 migration

## 核心原則

### Code 是 Liability

每一行 code 都有持續成本：要寫 test、寫文件、套 security patch、更新 dependency，並且周邊的人都要付出心智負擔。Code 的價值在它提供的功能，而不是 code 本身。當同樣的功能可以用更少 code、更少複雜度、更好抽象提供時 —— 舊的 code 就該走。

### Hyrum's Law 讓移除變難

只要使用者夠多，每個可觀察的行為都會被某人依賴 —— 包含 bug、timing 怪癖、未文件化的副作用。這就是為什麼 deprecation 需要主動 migration，不是只發公告就好。當使用者依賴的是替代品沒複製到的行為時，他們無法「直接切換」。

### Deprecation 規劃從設計時就開始

造新東西時要問：「3 年後我要怎麼把這拿掉？」具備乾淨 interface、feature flag、最小表面積的系統，比把實作細節到處外洩的系統好 deprecate 太多。

## Deprecation 決策

deprecate 任何東西前，先回答這些問題：

```
1. Does this system still provide unique value?
   → If yes, maintain it. If no, proceed.

2. How many users/consumers depend on it?
   → Quantify the migration scope.

3. Does a replacement exist?
   → If no, build the replacement first. Don't deprecate without an alternative.

4. What's the migration cost for each consumer?
   → If trivially automated, do it. If manual and high-effort, weigh against maintenance cost.

5. What's the ongoing maintenance cost of NOT deprecating?
   → Security risk, engineer time, opportunity cost of complexity.
```

## 強制式 vs 建議式 Deprecation

| 類型 | 何時使用 | 機制 |
|------|-------------|-----------|
| **建議式（Advisory）** | migration 是可選的、舊系統穩定 | warning、文件、提示。使用者照自己節奏遷移。 |
| **強制式（Compulsory）** | 舊系統有 security 問題、阻擋進度、或維護成本不可持續 | 硬 deadline。舊系統會在 X 日被移除。提供 migration 工具。 |

**預設為建議式。** 只有在維護成本或風險真的高到值得強迫遷移時才用強制式。強制式 deprecation 必須提供 migration 工具、文件與支援 —— 不能只丟一個 deadline 就走人。

## Migration 流程

### Step 1：先把替代品建好

沒有可用替代品就不要 deprecate。替代品必須：

- 涵蓋舊系統所有關鍵 use case
- 有文件與 migration guide
- 在 production 已驗證過（不只是「理論上更好」）

### Step 2：公告與文件化

```markdown
## Deprecation Notice: OldService

**Status:** Deprecated as of 2025-03-01
**Replacement:** NewService (see migration guide below)
**Removal date:** Advisory — no hard deadline yet
**Reason:** OldService requires manual scaling and lacks observability.
            NewService handles both automatically.

### Migration Guide
1. Replace `import { client } from 'old-service'` with `import { client } from 'new-service'`
2. Update configuration (see examples below)
3. Run the migration verification script: `npx migrate-check`
```

### Step 3：增量遷移

一次遷一個 consumer，不要一次全遷。對每個 consumer：

```
1. Identify all touchpoints with the deprecated system
2. Update to use the replacement
3. Verify behavior matches (tests, integration checks)
4. Remove references to the old system
5. Confirm no regressions
```

**Churn 規則：**如果你擁有正在被 deprecate 的基礎設施，你就有責任幫你的使用者遷移 —— 或提供向後相容、不需任何遷移動作的更新。不要公告 deprecation 然後丟使用者自己想辦法。

### Step 4：移除舊系統

只在所有 consumer 都遷完後才動：

```
1. Verify zero active usage (metrics, logs, dependency analysis)
2. Remove the code
3. Remove associated tests, documentation, and configuration
4. Remove the deprecation notices
5. Celebrate — removing code is an achievement
```

## Migration Patterns

### Strangler Pattern

舊新並存。逐步把流量從舊導到新。當舊系統處理 0% 的流量時就移除它。

```
Phase 1: New system handles 0%, old handles 100%
Phase 2: New system handles 10% (canary)
Phase 3: New system handles 50%
Phase 4: New system handles 100%, old system idle
Phase 5: Remove old system
```

### Adapter Pattern

寫一個 adapter，把舊 interface 的 call 翻譯到新實作。Consumer 仍用舊 interface，你後端做 migration。

```typescript
// Adapter: old interface, new implementation
class LegacyTaskService implements OldTaskAPI {
  constructor(private newService: NewTaskService) {}

  // Old method signature, delegates to new implementation
  getTask(id: number): OldTask {
    const task = this.newService.findById(String(id));
    return this.toOldFormat(task);
  }
}
```

### Feature Flag Migration

用 feature flag 一次切換一個 consumer 從舊到新：

```typescript
function getTaskService(userId: string): TaskService {
  if (featureFlags.isEnabled('new-task-service', { userId })) {
    return new NewTaskService();
  }
  return new LegacyTaskService();
}
```

## Zombie Code

Zombie code 是「沒人擁有但所有人都依賴」的 code。沒在主動維護、沒明確 owner、累積 security vulnerability 與相容性問題。徵兆：

- 6+ 個月沒 commit，但仍有 active consumer
- 沒有指派 maintainer 或 team
- test 失敗但沒人修
- dependency 有已知 vulnerability 但沒人更新
- 文件引用到已不存在的系統

**因應：**要嘛指派 owner 並好好維護，要嘛配上具體 migration 計畫一起 deprecate。Zombie code 不能停留在曖昧地帶 —— 不是給投資就是給移除。

## 常見合理化（Rationalizations）

| 合理化說法 | 實際情況 |
|---|---|
| 「它還能跑啊，幹嘛拿掉？」 | 沒人維護的 code 會默默累積 security debt 與複雜度。維護成本悄悄成長。 |
| 「之後可能有人會用到」 | 之後真的需要再重建。把沒在用的 code 留著「以備不時之需」比重建還貴。 |
| 「migration 太花錢了」 | 把 migration 成本跟 2-3 年的持續維護成本比一比。長期看 migration 通常更便宜。 |
| 「等新系統做完再 deprecate」 | Deprecation 規劃應在設計時開始。等新系統做完，你就有新優先級了。現在就規劃。 |
| 「使用者會自己遷」 | 不會。提供工具、文件、誘因 —— 或者自己幫他們遷（Churn 規則）。 |
| 「我們可以無限期同時維護兩套」 | 兩套做同件事 = 雙倍維護、雙倍 test、雙倍文件、雙倍 onboarding 成本。 |

## Red Flags

- Deprecate 但沒有可用替代品
- Deprecation 公告但沒有 migration 工具或文件
- 「軟」deprecation 維持建議式狀態多年沒進度
- Zombie code 沒 owner，但有 active consumer
- 在已 deprecate 的系統上加新功能（投資應放替代品）
- 沒量測現有 usage 就 deprecate
- 沒驗證有沒有人在用就把 code 移掉

## 驗證

完成一次 deprecation 後：

- [ ] 替代品已 production 驗證，涵蓋所有關鍵 use case
- [ ] migration guide 存在，有具體步驟與範例
- [ ] 所有 active consumer 已遷移（metrics/log 已驗證）
- [ ] 舊 code、test、文件、設定全部移除
- [ ] codebase 內已無對 deprecated 系統的引用
- [ ] deprecation 公告已移除（任務完成）
