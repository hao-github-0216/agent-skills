---
name: spec-driven-development
description: 在寫程式或跑 experiment 之前先寫 spec。當開新專案、新功能、新 ML experiment 或重大改動且尚未有 spec 時使用；當需求不清楚、模糊、或只是個粗糙想法時使用。English: Creates specs before coding or running experiments. Use when starting a new project, feature, ML experiment, or significant change and no spec exists yet. Use when requirements are unclear, ambiguous, or only exist as a vague idea.
---

# Spec-Driven Development

## 概覽

在寫任何程式或啟動任何 experiment 之前，先寫一份結構化的 specification。Spec 是共同的事實來源 —— 它定義我們在做什麼（或在驗證什麼假設）、為什麼做、以及怎麼判定完成。沒有 spec 的程式碼是猜測；沒有 spec 的 experiment 是在浪費 compute。

## Spec 在這個 fork 的領域中放在哪裡

軟體變更：spec 是給人類審閱的文件，**不**進 repo（repo 保持乾淨、不放 `.md` 紀錄 —— 見根目錄 CLAUDE.md）。

ML experiment 專屬：spec **就是** vault cell。每次 experiment run 一個 cell，路徑在：

```
~/vault/<Project>/Experiments/<tag>.md
```

Vault cell 是結構化的紀錄，包含 status enum（planned → running → done/failed）、config dump、hypothesis、預估 runtime、partition（sanity / ampere / dgxh）、result 連結，以及跑完後的筆記。它在 submit **之前**就存在，這正是讓一次 run 成為刻意 experiment 而不是 yolo 的關鍵。`experiment-vault` skill 涵蓋 cell 範本 —— 格式細節以該 skill 為準。

**不要**把 experiment spec／結果／決策記錄當成 `.md` 檔放進 GitHub repo。Vault 才是正典 journal，repo 保持乾淨。

## 何時使用

- 開新專案或新功能
- 需求模糊或不完整
- 改動牽涉到多個檔案或 module
- 即將做架構決策
- 預期實作會超過 30 分鐘

**何時不用：** 一行修改、錯字修正，或需求毫無歧義且自我完備的變更。

## Gated Workflow

Spec-driven development 有四個階段。在當前階段尚未驗證通過前，不要進到下一階段。

```
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT
   │          │        │          │
   ▼          ▼        ▼          ▼
 Human      Human    Human      Human
 reviews    reviews  reviews    reviews
```

### 階段 1：Specify

從高層次的願景開始。向人類提出釐清問題，直到需求變得具體。

**立刻把假設攤開來。** 在開始寫 spec 內容之前，列出你正在假設的事項：

```
ASSUMPTIONS I'M MAKING:
1. This is a web application (not native mobile)
2. Authentication uses session-based cookies (not JWT)
3. The database is PostgreSQL (based on existing Prisma schema)
4. We're targeting modern browsers only (no IE11)
→ Correct me now or I'll proceed with these.
```

不要默默地把模糊需求補完。Spec 的全部用途就是在程式碼寫下去**之前**揭露誤解 —— 而假設正是最危險的誤解形式。

**寫一份 spec 文件，涵蓋這六大核心區塊：**

1. **Objective** —— 我們在做什麼，為什麼？User 是誰？成功的樣子是什麼？

2. **Commands** —— 完整可執行的指令含 flag，不只是工具名稱。
   ```
   Build: npm run build
   Test: npm test -- --coverage
   Lint: npm run lint --fix
   Dev: npm run dev
   ```

3. **Project Structure** —— 原始碼放哪、test 放哪、文件放哪。
   ```
   src/           → Application source code
   src/components → React components
   src/lib        → Shared utilities
   tests/         → Unit and integration tests
   e2e/           → End-to-end tests
   docs/          → Documentation
   ```

4. **Code Style** —— 一段真的程式碼片段勝過三段文字描述。包含命名規範、格式規則，以及好輸出的範例。

5. **Testing Strategy** —— 用什麼 framework、test 放哪、coverage 預期、哪一層 test 處理哪種關注點。

6. **Boundaries** —— 三層制：
   - **Always do:** commit 前跑 test、遵守命名規範、驗證 input
   - **Ask first:** 改 database schema、加 dependency、改 CI 設定
   - **Never do:** commit secrets、編輯 vendor 目錄、未經核准刪掉 failing test

**Spec 範本：**

```markdown
# Spec: [Project/Feature Name]

## Objective
[What we're building and why. User stories or acceptance criteria.]

## Tech Stack
[Framework, language, key dependencies with versions]

## Commands
[Build, test, lint, dev — full commands]

## Project Structure
[Directory layout with descriptions]

## Code Style
[Example snippet + key conventions]

## Testing Strategy
[Framework, test locations, coverage requirements, test levels]

## Boundaries
- Always: [...]
- Ask first: [...]
- Never: [...]

## Success Criteria
[How we'll know this is done — specific, testable conditions]

## Open Questions
[Anything unresolved that needs human input]
```

**把指令重新詮釋為成功條件。** 收到模糊需求時，把它翻譯成具體的條件：

```
REQUIREMENT: "Make the dashboard faster"

REFRAMED SUCCESS CRITERIA:
- Dashboard LCP < 2.5s on 4G connection
- Initial data load completes in < 500ms
- No layout shift during load (CLS < 0.1)
→ Are these the right targets?
```

這讓你能朝著明確目標 loop、retry、解問題，而不是猜「更快」是什麼意思。

### 階段 2：Plan

拿到驗證過的 spec 後，產出技術實作計畫：

1. 找出主要元件以及它們的相依關係
2. 決定實作順序（什麼必須先建）
3. 標記風險與應對策略
4. 找出哪些可以並行、哪些必須序列
5. 在階段之間定義驗證檢查點

Plan 應該是可審閱的：人類看完應該能說「對，這個方向沒問題」或「不對，改 X」。

### 階段 3：Tasks

把計畫拆成離散的、可實作的 task：

- 每個 task 應該能在一個專注的工作段落內完成
- 每個 task 都有明確的 acceptance criteria
- 每個 task 都包含一個驗證步驟（test、build、手動檢查）
- Task 依相依性排序，不是依「感覺重要程度」
- 沒有 task 應該動到超過 ~5 個檔案

**Task 範本：**
```markdown
- [ ] Task: [Description]
  - Acceptance: [What must be true when done]
  - Verify: [How to confirm — test command, build, manual check]
  - Files: [Which files will be touched]
```

### 階段 4：Implement

依照 `incremental-implementation` 與 `test-driven-development` skill，一次執行一個 task。用 `context-engineering` 在每一步載入正確的 spec 段落與原始檔，而不是把整份 spec 一次塞給 agent。

## 讓 Spec 保持活著

Spec 是活文件，不是一次性產物：

- **決策變更時就更新** —— 如果發現 data model 要改，先改 spec，再實作。
- **範圍變更時就更新** —— 加了或砍了功能都應該反映在 spec 上。
- **把 spec 進版控** —— Spec 跟程式碼一起放在 version control。
- **PR 中引用 spec** —— 連回該 PR 實作的 spec 段落。

## 常見的合理化說法

| Rationalization | Reality |
|---|---|
| 「這很簡單，不用 spec」 | 簡單 task 不需要*長* spec，但仍需要 acceptance criteria。兩行的 spec 也可以。 |
| 「我寫完 code 再寫 spec」 | 那是文件，不是 specification。Spec 的價值在於**事前**強迫你想清楚。 |
| 「Spec 會拖慢進度」 | 15 分鐘的 spec 省下幾小時的 rework。15 分鐘的 waterfall 勝過 15 小時的 debug。 |
| 「需求反正會變」 | 所以 spec 才是活文件。過時的 spec 仍然比沒有 spec 好。 |
| 「使用者知道自己要什麼」 | 連明確的需求都有隱含假設。Spec 的功能就是把那些假設攤開。 |

## Red Flags

- 沒有任何書面需求就開始寫 code
- 在釐清「完成」是什麼意思之前就問「我可以開始 build 了嗎？」
- 實作了任何 spec／task list 上沒提到的功能
- 做架構決策但沒記錄
- 跳過 spec 因為「要做什麼很明顯」

## Verification

進入實作之前確認：

- [ ] Spec 涵蓋全部六大核心區塊
- [ ] 人類已審閱並核准 spec
- [ ] 成功條件明確且可測試
- [ ] Boundaries（Always／Ask First／Never）已定義
- [ ] Spec 已存成 repo 中的檔案
