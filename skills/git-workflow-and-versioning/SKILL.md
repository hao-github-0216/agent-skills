---
name: git-workflow-and-versioning
description: 建立 git workflow 規範。任何程式碼變更都適用。當需要 commit、開 branch、解 conflict，或需要組織多條平行工作流時使用。English: Structures git workflow practices. Use when making any code change. Use when committing, branching, resolving conflicts, or when you need to organize work across multiple parallel streams.
---

# Git Workflow and Versioning

## Overview

Git 是你的安全網。把 commit 當作存檔點、branch 當作沙盒、history 當作文件。當 AI agent 以高速產出程式碼時，有紀律的版本控制就是讓變更維持可控、可審查、可回復的核心機制。

在這個 fork 的領域裡，git 還扮演著版本控制以外的角色：**它是機器之間的資料傳輸通道。**

## When to Use

永遠都要用。每一次程式碼變更都會經過 git。這個環境下所有跨裝置的資料傳輸也都會經過 git — 結果絕對不要用 `scp` / `rsync`。

## Git as data transport — the Bridge2HPC pattern

這個 fork 的擁有者在 OSU HPC 上跑實驗，並在 howardserver 上看結果。兩者之間的傳輸通道就是 GitHub：

```
howardserver edit
    ─ git push ─▶  GitHub (origin)
                    │
                    └─ git pull (HPC, manual or via Bridge2HPC submit.sh)
                       │
                       SLURM job runs
                       │
                       └─ at job end:  cd repo && git add results/ && git commit && git push
                                         │
                                         └─ howardserver cron runs ~/Bridge2HPC/howardserver/poll.sh
                                            every 2 min → git pull → results land in
                                            ~/<repo>/local_artifacts/<tag>/
```

**這對 commit 衛生的影響：**

- HPC 端的 commit message 是你幾週後回頭追溯這次 run 做了什麼的依據。請包含：run tag、partition、exit code、關鍵指標（例如 `[run/poison-eps8] dgxh, exit 0, ASR=0.34`）。
- `results/` 或 `local_artifacts/` 目錄雖然是「輸出」，但故意被 git 追蹤 — 這就是這裡資料傳輸的方式。本使用者的 repo 中 `.gitignore` 不應該擋掉這些路徑。
- 在研究 repo 上對 main force-push 會抹掉 HPC job 進行中產出的 commit。**只要還有 SLURM job 正在寫入這個 repo，就絕對不要 `git push --force`。**
- 如果 `poll.sh` 回報「沒有新 commit」但 job 應該已經跑完，去檢查 HPC 上的 clone：`ssh -J flip submit "cd <repo> && git log --oneline -5"`。可能是 job 在 commit 之前就 crash，或 push 默默失敗。

## Repo policy specific to this fork

- **repo 內不放 `.md` 決策日誌 / 實驗筆記。** 位於 `~/vault/<Project>/` 的 vault 才是日誌所在。Repo 只給程式碼用。如果你發現自己想 commit `EXPERIMENTS.md` 或 `NOTES.md`，請改成寫進 vault cell。
- **絕對不要為了個人客製而開長期存活的 per-machine branch。** 一條 `main`，由 howardserver / HPC / Mac 共用。（per-machine 差異住在 repo 之外：`~/.claude/skills/` 或 dotfiles。）`agent-skills-fork-dev` 這個 user-scope skill 把這條規則編碼了。

## Core Principles

### Trunk-Based Development (Recommended)

讓 `main` 隨時都能部署。在短期存活的 feature branch 上工作，並在 1-3 天內 merge 回來。長期存活的 development branch 是隱藏成本 — 它們會發散、製造 merge conflict、延遲整合。DORA 的研究一再顯示 trunk-based development 與高績效工程團隊有正相關。

```
main ──●──●──●──●──●──●──●──●──●──  (always deployable)
        ╲      ╱  ╲    ╱
         ●──●─╱    ●──╱    ← short-lived feature branches (1-3 days)
```

這是建議的預設策略。使用 gitflow 或長期存活 branch 的團隊可以把這些原則（atomic commit、小變更、有描述性的 message）套到自己的 branching model 上 — commit 紀律比特定的 branching 策略更重要。

- **Dev branch 是成本。** branch 每多活一天，就多累積一天的 merge 風險。
- **Release branch 可以接受。** 當你需要在 main 繼續推進的同時穩定一個 release。
- **Feature flag > 長期 branch。** 寧可把未完成的工作部署在 flag 後面，也不要讓它在 branch 上躺好幾週。

### 1. Commit Early, Commit Often

每一個成功的增量都要有自己的 commit。不要累積大量未 commit 的變更。

```
Work pattern:
  Implement slice → Test → Verify → Commit → Next slice

Not this:
  Implement everything → Hope it works → Giant commit
```

Commit 是存檔點。如果下一次變更壞了什麼，你可以瞬間回到上一個已知正常的狀態。

### 2. Atomic Commits

每個 commit 只做一件邏輯上的事：

```
# Good: Each commit is self-contained
git log --oneline
a1b2c3d Add task creation endpoint with validation
d4e5f6g Add task creation form component
h7i8j9k Connect form to API and add loading state
m1n2o3p Add task creation tests (unit + integration)

# Bad: Everything mixed together
git log --oneline
x1y2z3a Add task feature, fix sidebar, update deps, refactor utils
```

### 3. Descriptive Messages

Commit message 要解釋 *為什麼*，而不只是 *做了什麼*：

```
# Good: Explains intent
feat: add email validation to registration endpoint

Prevents invalid email formats from reaching the database.
Uses Zod schema validation at the route handler level,
consistent with existing validation patterns in auth.ts.

# Bad: Describes what's obvious from the diff
update auth.ts
```

**格式：**
```
<type>: <short description>

<optional body explaining why, not what>
```

**Types：**
- `feat` — 新功能
- `fix` — bug 修復
- `refactor` — 既不修 bug 也不加功能的程式碼變更
- `test` — 新增或更新測試
- `docs` — 只改文件
- `chore` — 工具、相依套件、設定

### 4. Keep Concerns Separate

不要把格式變更跟行為變更混在一起。不要把 refactor 跟 feature 混在一起。每一種變更都應該是獨立的 commit — 理想上也是獨立的 PR：

```
# Good: Separate concerns
git commit -m "refactor: extract validation logic to shared utility"
git commit -m "feat: add phone number validation to registration"

# Bad: Mixed concerns
git commit -m "refactor validation and add phone number field"
```

**把 refactor 跟 feature work 分開。** 一次 refactor 跟一次 feature 是兩個不同的變更 — 分別送出。這讓每個變更更容易 review、revert，也更容易在歷史紀錄裡理解。小型清理（例如重新命名變數）可在 reviewer 同意下合進 feature commit。

### 5. Size Your Changes

每個 commit/PR 目標約 100 行。超過約 1000 行的變更應該拆開。拆解大變更的策略可以參考 `code-review-and-quality`。

```
~100 lines  → Easy to review, easy to revert
~300 lines  → Acceptable for a single logical change
~1000 lines → Split into smaller changes
```

## Branching Strategy

### Feature Branches

```
main (always deployable)
  │
  ├── feature/task-creation    ← One feature per branch
  ├── feature/user-settings    ← Parallel work
  └── fix/duplicate-tasks      ← Bug fixes
```

- 從 `main`（或團隊的預設 branch）開出來
- branch 要短期存活（1-3 天內 merge）— 長期 branch 是隱藏成本
- merge 完就刪 branch
- 對未完成的功能，優先採用 feature flag 而不是長期 branch

### Branch Naming

```
feature/<short-description>   → feature/task-creation
fix/<short-description>       → fix/duplicate-tasks
chore/<short-description>     → chore/update-deps
refactor/<short-description>  → refactor/auth-module
```

## Working with Worktrees

對於平行的 AI agent 工作，使用 git worktree 同時跑多個 branch：

```bash
# Create a worktree for a feature branch
git worktree add ../project-feature-a feature/task-creation
git worktree add ../project-feature-b feature/user-settings

# Each worktree is a separate directory with its own branch
# Agents can work in parallel without interfering
ls ../
  project/              ← main branch
  project-feature-a/    ← task-creation branch
  project-feature-b/    ← user-settings branch

# When done, merge and clean up
git worktree remove ../project-feature-a
```

優點：
- 多個 agent 可同時在不同 feature 上工作
- 不需要切 branch（每個目錄各自有自己的 branch）
- 任何一個實驗失敗就直接刪 worktree — 不會丟失任何東西
- 變更在被明確 merge 之前都是隔離的

## The Save Point Pattern

```
Agent starts work
    │
    ├── Makes a change
    │   ├── Test passes? → Commit → Continue
    │   └── Test fails? → Revert to last commit → Investigate
    │
    ├── Makes another change
    │   ├── Test passes? → Commit → Continue
    │   └── Test fails? → Revert to last commit → Investigate
    │
    └── Feature complete → All commits form a clean history
```

這個模式代表你最多只會丟失一個增量的工作。如果 agent 偏離正軌，`git reset --hard HEAD` 會把你帶回最後一個成功的狀態。

## Change Summaries

任何修改之後，給一份結構化的摘要。這讓 review 更輕鬆、能記錄 scope 紀律、也會把意外的變更浮上來：

```
CHANGES MADE:
- src/routes/tasks.ts: Added validation middleware to POST endpoint
- src/lib/validation.ts: Added TaskCreateSchema using Zod

THINGS I DIDN'T TOUCH (intentionally):
- src/routes/auth.ts: Has similar validation gap but out of scope
- src/middleware/error.ts: Error format could be improved (separate task)

POTENTIAL CONCERNS:
- The Zod schema is strict — rejects extra fields. Confirm this is desired.
- Added zod as a dependency (72KB gzipped) — already in package.json
```

這個模式可以提早抓到錯誤的假設，並給 reviewer 一張清楚的變更地圖。「DIDN'T TOUCH」那段尤其重要 — 它顯示你有 scope 紀律，沒有跑去做沒人要的整修。

## Pre-Commit Hygiene

每次 commit 前：

```bash
# 1. Check what you're about to commit
git diff --staged

# 2. Ensure no secrets
git diff --staged | grep -i "password\|secret\|api_key\|token"

# 3. Run tests
npm test

# 4. Run linting
npm run lint

# 5. Run type checking
npx tsc --noEmit
```

用 git hook 自動化：

```json
// package.json (using lint-staged + husky)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

## Handling Generated Files

- **Commit 產生的檔案** 只在專案預期會有它們時才做（例如 `package-lock.json`、Prisma migration）
- **不要 commit** build 輸出（`dist/`、`.next/`）、環境檔（`.env`）、或 IDE 設定（`.vscode/settings.json` 除非是共用的）
- **要有 `.gitignore`** 涵蓋：`node_modules/`、`dist/`、`.env`、`.env.local`、`*.pem`

## Using Git for Debugging

```bash
# Find which commit introduced a bug
git bisect start
git bisect bad HEAD
git bisect good <known-good-commit>
# Git checkouts midpoints; run your test at each to narrow down

# View what changed recently
git log --oneline -20
git diff HEAD~5..HEAD -- src/

# Find who last changed a specific line
git blame src/services/task.ts

# Search commit messages for a keyword
git log --grep="validation" --oneline
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「我等 feature 做完再 commit」 | 一個巨大的 commit 沒辦法 review、debug、或 revert。每個切片都要 commit。 |
| 「message 不重要」 | message 是文件。未來的你（與未來的 agent）需要理解改了什麼、為什麼改。 |
| 「我之後會全部 squash」 | squash 會破壞開發過程的敘事。從一開始就保持乾淨的增量 commit 比較好。 |
| 「branch 是額外負擔」 | 短期 branch 是免費的，可以避免有衝突的工作互撞。長期 branch 才是問題 — 1-3 天內就要 merge。 |
| 「這個變更我之後再拆」 | 大變更難 review、部署風險高、也難 revert。在送出前就拆，不是事後拆。 |
| 「我不需要 .gitignore」 | 直到 `.env` 含正式環境的 secret 被 commit 上去那一刻。立刻設定好。 |

## Red Flags

- 大量未 commit 的變更累積中
- commit message 像「fix」、「update」、「misc」
- 格式變更跟行為變更混在一起
- 專案沒有 `.gitignore`
- commit 了 `node_modules/`、`.env`、或 build 產物
- 長期存活、跟 main 大幅發散的 branch
- 對共用 branch 做 force-push

## Verification

每個 commit：

- [ ] commit 只做一件邏輯上的事
- [ ] message 說明了 why，並遵守 type 規範
- [ ] commit 前 test 都通過
- [ ] diff 裡沒有 secret
- [ ] 沒有把純格式變更跟行為變更混在一起
- [ ] `.gitignore` 涵蓋了標準排除項
