---
name: code-review-and-quality
description: 進行多面向的 code review。用於 merge 任何變更之前。用於 review 由你自己、其他 agent 或人類所寫的程式碼。用於變更進入 main branch 之前需要從多個面向評估程式碼品質時。English: Conducts multi-axis code review. Use before merging any change. Use when reviewing code written by yourself, another agent, or a human. Use when you need to assess code quality across multiple dimensions before it enters the main branch.
---

# Code Review and Quality

## Overview

帶有品質門檻的多面向 code review。每個變更在 merge 前都要經過 review——沒有例外。Review 從五個面向著手：correctness、readability、architecture、security 與 performance。

**Approve 的標準：** 當一個變更明確改善整體程式碼健康度時就 approve，即使它不完美。完美的程式碼並不存在——目標是持續改進。不要因為它沒寫成你的習慣就 block。如果它讓 codebase 變好且符合專案慣例，就 approve。

## When to Use

- 任何 PR 或變更 merge 之前
- 完成一個功能的實作之後
- 當另一個 agent 或 model 產出你需要評估的程式碼時
- 重構既有程式碼時
- 任何 bug fix 之後（review fix 與 regression test 兩者）

## The Five-Axis Review

每次 review 都從以下面向評估程式碼：

### 1. Correctness

這段程式碼有做到它聲稱的事嗎？

- 它符合 spec 或任務需求嗎？
- Edge case 有處理嗎（null、空值、邊界值）？
- 錯誤路徑有處理嗎（不只是 happy path）？
- 全部測試都過嗎？這些測試是真的在測對的東西嗎？
- 有沒有 off-by-one 錯誤、race condition 或狀態不一致？

### 2. Readability & Simplicity

別的工程師（或 agent）能不靠作者解釋就看懂這段程式碼嗎？

- 命名有描述性、與專案慣例一致嗎？（不要出現沒上下文的 `temp`、`data`、`result`）
- 控制流是否簡單明瞭（避免巢狀 ternary、深層 callback）？
- 程式碼組織是否合邏輯（相關程式碼成組、模組邊界清晰）？
- 有沒有應該被簡化的「炫技」寫法？
- **這能不能用更少行數寫完？** （該 100 行解決卻寫 1000 行就是失敗）
- **abstraction 是否物有所值？** （在出現第三個 use case 之前不要泛化）
- 有沒有 comment 能釐清不明顯的意圖？（但別對顯而易見的程式碼加 comment）
- 有沒有死掉的程式碼遺骸：no-op 變數（`_unused`）、向後相容的 shim、或 `// removed` comment？

### 3. Architecture

這個變更符合系統設計嗎？

- 它沿用現有 pattern，還是引入新的 pattern？若引入新的，理由充分嗎？
- 它是否維持乾淨的模組邊界？
- 有沒有應該共用的程式碼重複？
- Dependency 流向是否正確（沒有 circular dependency）？
- abstraction 層級是否合適（沒有 over-engineer、也沒太過耦合）？

### 4. Security

詳細的安全指引請見 `security-and-hardening`。這個變更會引入漏洞嗎？

- 使用者輸入有 validate 與 sanitize 嗎？
- Secret 是否被排除在程式碼、log 與版本控制之外？
- 在需要的地方有檢查 authentication/authorization 嗎？
- SQL 查詢有用 parameter 嗎（不是字串拼接）？
- 輸出是否經過 encode 以防 XSS？
- Dependency 是否來自可信來源、沒有已知漏洞？
- 來自外部來源（API、log、使用者內容、設定檔）的資料有被當成不可信任嗎？
- 外部資料流在用於邏輯或渲染前，是否在系統邊界做 validation？

### 5. Performance

詳細的 profiling 與優化請見 `performance-optimization`。這個變更會引入效能問題嗎？

- 有沒有 N+1 query pattern？
- 有沒有 unbounded 的 loop 或不受限制的資料抓取？
- 有沒有應該改成 async 的同步操作？
- UI component 是否有不必要的 re-render？
- List endpoint 是否漏掉 pagination？
- Hot path 是否建立大型物件？

## Change Sizing

小而專注的變更比較好 review、merge 比較快、deploy 比較安全。目標大小：

```
~100 lines changed   → Good. Reviewable in one sitting.
~300 lines changed   → Acceptable if it's a single logical change.
~1000 lines changed  → Too large. Split it.
```

**何謂「一個變更」：** 一個自我包含的修改，只處理一件事、附帶相關測試，並讓系統在送出後仍可運作。它是某個功能的一部分——不是整個功能。

**變更過大時的拆分策略：**

| Strategy | How | When |
|----------|-----|------|
| **Stack** | 先送出小變更，下一個變更基於它 | 有先後依賴關係 |
| **By file group** | 對需要不同 reviewer 的檔案群分別變更 | 跨越多個關注點 |
| **Horizontal** | 先建立共用程式碼/stub，再做 consumer | 分層架構 |
| **Vertical** | 拆成更小的全端切片 | 功能開發 |

**大型變更可被接受的情形：** 整個檔案的刪除、自動化的 refactor，這類 reviewer 只需確認意圖、不必逐行驗證的情況。

**把 refactor 與功能開發分開。** 同時 refactor 既有程式碼又新增行為的變更其實是兩個變更——分開送出。小型清理（變數改名）可以 reviewer 自由裁量是否一起。

## Change Descriptions

每個變更都需要一段在版本歷史中能獨立站得住的描述。

**第一行：** 簡短、祈使句、可獨立閱讀。「Delete the FizzBuzz RPC」而不是「Deleting the FizzBuzz RPC」。資訊量必須足以讓在歷史中搜尋的人不必看 diff 也能理解這個變更。

**主體：** 改了什麼、為什麼。包含程式碼本身看不到的脈絡、決策與推理。在合適時連結到 bug 編號、benchmark 結果或 design doc。當作法有缺陷時要承認。

**反模式：** 「Fix bug」「Fix build」「Add patch」「Moving code from A to B」「Phase 1」「Add convenience functions」。

## Review Process

### Step 1: Understand the Context

在看程式碼之前先理解意圖：

```
- What is this change trying to accomplish?
- What spec or task does it implement?
- What is the expected behavior change?
```

### Step 2: Review the Tests First

測試會透露意圖與覆蓋率：

```
- Do tests exist for the change?
- Do they test behavior (not implementation details)?
- Are edge cases covered?
- Do tests have descriptive names?
- Would the tests catch a regression if the code changed?
```

### Step 3: Review the Implementation

帶著五個面向逐一檢視程式碼：

```
For each file changed:
1. Correctness: Does this code do what the test says it should?
2. Readability: Can I understand this without help?
3. Architecture: Does this fit the system?
4. Security: Any vulnerabilities?
5. Performance: Any bottlenecks?
```

### Step 4: Categorize Findings

替每個 comment 標上嚴重度，讓作者知道哪些是必修、哪些是可選：

| Prefix | Meaning | Author Action |
|--------|---------|---------------|
| *(no prefix)* | 必修 | merge 前必須處理 |
| **Critical:** | block merge | 安全漏洞、資料遺失、功能壞掉 |
| **Nit:** | 微小、可選 | 作者可以忽略——格式、風格偏好 |
| **Optional:** / **Consider:** | 建議 | 值得考慮但非必要 |
| **FYI** | 純資訊 | 不需動作——供未來參考的脈絡 |

這能避免作者把所有意見都當必修，浪費時間在可選建議上。

### Step 5: Verify the Verification

檢查作者的驗證說明：

```
- What tests were run?
- Did the build pass?
- Was the change tested manually?
- Are there screenshots for UI changes?
- Is there a before/after comparison?
```

## Multi-Model Review Pattern

用不同 model 提供不同 review 觀點：

```
Model A writes the code
    │
    ▼
Model B reviews for correctness and architecture
    │
    ▼
Model A addresses the feedback
    │
    ▼
Human makes the final call
```

這能補捉單一 model 可能漏掉的問題——不同 model 有不同盲點。

**review agent 的範例 prompt：**
```
Review this code change for correctness, security, and adherence to
our project conventions. The spec says [X]. The change should [Y].
Flag any issues as Critical, Important, or Suggestion.
```

## Dead Code Hygiene

任何 refactor 或實作變更後，檢查孤兒程式碼：

1. 找出現在已經到不了或沒被使用的程式碼
2. 明確列出來
3. **刪除前先問：** 「我可以移除這些已經沒被用到的東西嗎：[列表]？」

不要把死程式碼留下——它會誤導後續的讀者與 agent。但也不要在不確定時就靜悄悄刪掉。有疑慮時就問。

```
DEAD CODE IDENTIFIED:
- formatLegacyDate() in src/utils/date.ts — replaced by formatDate()
- OldTaskCard component in src/components/ — replaced by TaskCard
- LEGACY_API_URL constant in src/config.ts — no remaining references
→ Safe to remove these?
```

## Review Speed

慢的 review 會 block 整個團隊。切換脈絡來做 review 的成本，比強加在別人身上的等待成本還小。

- **一個工作日內回覆**——這是上限不是目標
- **理想節奏：** review 請求一進來就盡快回覆，除非正深陷在專注 coding 中。一個典型變更應該在一天內走完數輪 review
- **快速個別回覆優先於最終 approve 的速度**。即使要多輪，也比較不會讓人沮喪
- **大型變更：** 請作者拆分，而不是 review 一個巨大 changeset

## Handling Disagreements

處理 review 爭議時，套用以下優先順序：

1. **技術事實與資料** 凌駕主觀意見與偏好
2. **Style guide** 在風格議題上是絕對權威
3. **軟體設計** 必須以工程原則來評估，而非個人偏好
4. **codebase 一致性** 在不損害整體健康度的前提下可接受

**不要接受「我之後會清」。** 經驗顯示延後的清理很少真的發生。除非真的緊急，否則要求送出前完成清理。如果周邊問題無法在這次變更裡處理，要求建立一個指派給作者自己的 bug。

## Honesty in Review

在 review 程式碼時——不論作者是你、其他 agent 或人類：

- **不要走過場。** 沒有 review 證據的「LGTM」對誰都沒幫助。
- **不要軟化真實問題。** 明明會打到 production 的 bug 卻說「可能是個小疑慮」是不誠實。
- **能量化就量化問題。** 「這個 N+1 query 在 list 中每筆會多 ~50ms」勝過「這可能會慢」。
- **對明顯有問題的作法直接反駁。** 在 review 中討好是一種失敗模式。如果實作有問題，直接說、提出替代方案。
- **被覆蓋時優雅接受。** 如果作者掌握完整脈絡並不同意，尊重他的判斷。對程式碼 comment、不要對人——把個人批評重新框架成對程式碼本身的討論。

## Dependency Discipline

Code review 的一部分就是 dependency review：

**新增任何 dependency 之前：**
1. 現有的 stack 能解決嗎？（通常可以。）
2. 這個 dependency 有多大？（檢查 bundle 影響。）
3. 它是否有在維護？（看最後 commit、open issue。）
4. 有沒有已知漏洞？（`npm audit`）
5. License 是什麼？（必須與專案相容。）

**規則：** 優先使用 standard library 與既有 utility，而不是新增 dependency。每個 dependency 都是一個負債。

## The Review Checklist

```markdown
## Review: [PR/Change title]

### Context
- [ ] I understand what this change does and why

### Correctness
- [ ] Change matches spec/task requirements
- [ ] Edge cases handled
- [ ] Error paths handled
- [ ] Tests cover the change adequately

### Readability
- [ ] Names are clear and consistent
- [ ] Logic is straightforward
- [ ] No unnecessary complexity

### Architecture
- [ ] Follows existing patterns
- [ ] No unnecessary coupling or dependencies
- [ ] Appropriate abstraction level

### Security
- [ ] No secrets in code
- [ ] Input validated at boundaries
- [ ] No injection vulnerabilities
- [ ] Auth checks in place
- [ ] External data sources treated as untrusted

### Performance
- [ ] No N+1 patterns
- [ ] No unbounded operations
- [ ] Pagination on list endpoints

### Verification
- [ ] Tests pass
- [ ] Build succeeds
- [ ] Manual verification done (if applicable)

### Verdict
- [ ] **Approve** — Ready to merge
- [ ] **Request changes** — Issues must be addressed
```
## See Also

- 詳細的 security review 指引見 `references/security-checklist.md`
- 效能 review 檢查見 `references/performance-checklist.md`

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「能跑就好了」 | 能跑但難讀、不安全或架構錯誤的程式碼會累積成複利的負債。 |
| 「我自己寫的，我知道是對的」 | 作者對自己的假設是盲的。每個變更都會從另一雙眼睛中受益。 |
| 「之後再清理」 | 「之後」永遠不會來。Review 就是品質門檻——好好用它。要求 merge 前清理，不是 merge 後。 |
| 「AI 產生的程式碼大概沒事吧」 | AI 產的程式碼需要更多檢視，不是更少。它聽起來自信、看起來合理，即使是錯的。 |
| 「測試過了，就 OK 了」 | 測試是必要但不充分。它無法捕捉架構問題、安全議題或可讀性疑慮。 |

## Red Flags

- PR 沒有任何 review 就被 merge
- Review 只看測試是否通過（忽略其他面向）
- 沒有真正 review 證據的「LGTM」
- 安全敏感的變更沒有以安全為焦點的 review
- 「太大難以好好 review」的大型 PR（拆掉它）
- Bug fix PR 沒有附 regression test
- Review 意見沒有嚴重度標籤——讓人分不清哪些必修、哪些可選
- 接受「我之後再修」——這從不會發生

## Verification

Review 完成後：

- [ ] 所有 Critical 議題都已解決
- [ ] 所有 Important 議題已解決，或被明確延後並有合理理由
- [ ] 測試通過
- [ ] Build 成功
- [ ] 驗證說明已記錄（改了什麼、如何驗證）
