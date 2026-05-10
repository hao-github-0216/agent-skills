---
name: source-driven-development
description: 讓每一項實作決策都有官方文件作為依據。當你想要產出可信、附 source 引用、不含過時 pattern 的程式碼時使用。當你在使用任何「正確性很重要」的 framework 或 library 開發時也適用。English: Grounds every implementation decision in official documentation. Use when you want authoritative, source-cited code free from outdated patterns. Use when building with any framework or library where correctness matters.
---

# Source-Driven Development

## Overview

每個 framework-specific 的程式碼決策都必須有 official documentation 作為依據。不要憑記憶實作 — 要 verify、cite，並讓使用者看到你的 source。Training data 會過期、API 會被 deprecated、最佳實務也會演進。這個 skill 確保使用者拿到的是值得信任的程式碼，因為每個 pattern 都能追溯到一份他們可以查證的權威 source。

## ML / CUDA stack — version pinning 不可妥協

在這個 fork 的領域中，「framework-specific」大多指的是 **PyTorch ↔ CUDA ↔ cuDNN ↔ GPU compute capability** 這個 stack。這個矩陣毫不留情，silent failure 是常態而非例外。在建議任何 config 之前永遠先驗證：

| What to check | Why | How |
|---|---|---|
| PyTorch 版本 vs. CUDA build | `torch==1.13+cu117` ≠ `torch==2.1+cu118`，混用會導致 link error / NaN | `python -c "import torch; print(torch.__version__, torch.version.cuda)"` |
| GPU compute capability vs. PyTorch 編譯支援的 arch | **H100 (sm_90) 在 PyTorch < 2.0 / cu117 上會 silently NaN** — 已確認的 gotcha | `nvidia-smi --query-gpu=compute_cap --format=csv`，再用 `torch.cuda.get_arch_list()` 檢查 |
| 你需要的 op 是否有 cuDNN deterministic-mode | 部分 op 沒有 deterministic kernel；`torch.use_deterministic_algorithms(True)` 會直接 crash | 查 PyTorch docs 中「Reproducibility」頁的對應 op |
| 你 dtype 的 mixed precision (`autocast`) 支援 | bf16 需要 Ampere+；fp16 各處都支援但有 overflow 風險 | PyTorch AMP docs |

**原則：** 在建議「升級 PyTorch」或「換 partition」之前，先到 [PyTorch previous-versions page](https://pytorch.org/get-started/previous-versions/)（或當前 install matrix）確認 GPU + driver + CUDA 的組合與目標 PyTorch 版本相容。不要憑記憶實作 — 矩陣每次 PyTorch release 都會變。

這也是為什麼 H100 sm_90 的 gotcha 被另外記錄在 `debugging-and-error-recovery` 裡：它是已被記錄的相容性失敗，不是神祕 bug。

## When to Use

- 使用者希望取得對應 framework 當前最佳實務的程式碼
- 在 build boilerplate、起始 code 或會在專案中被到處複製的 pattern
- 使用者明確要求「有文件依據」、「已驗證」或「正確」的實作
- 實作那些 framework 推薦做法很關鍵的功能（form、routing、data fetching、state management、auth）
- 在 review 或改進使用 framework-specific pattern 的程式碼
- 任何你即將憑記憶寫 framework-specific code 的場合

**不適用的情況：**

- 正確性不依賴於特定版本（rename 變數、修正 typo、搬檔案）
- 純邏輯，跨版本表現相同（迴圈、條件、資料結構）
- 使用者明確希望速度優先而非驗證優先（「快點做就好」）

## The Process

```
DETECT ──→ FETCH ──→ IMPLEMENT ──→ CITE
  │          │           │            │
  ▼          ▼           ▼            ▼
 What       Get the    Follow the   Show your
 stack?     relevant   documented   sources
            docs       patterns
```

### Step 1: Detect Stack and Versions

讀取專案的 dependency file 以辨識精確版本：

```
package.json    → Node/React/Vue/Angular/Svelte
composer.json   → PHP/Symfony/Laravel
requirements.txt / pyproject.toml → Python/Django/Flask
go.mod          → Go
Cargo.toml      → Rust
Gemfile         → Ruby/Rails
```

明確說出你發現了什麼：

```
STACK DETECTED:
- React 19.1.0 (from package.json)
- Vite 6.2.0
- Tailwind CSS 4.0.3
→ Fetching official docs for the relevant patterns.
```

如果版本缺失或模糊，**詢問使用者**。不要猜 — 版本決定了哪些 pattern 是正確的。

### Step 2: Fetch Official Documentation

抓取你正在實作的功能對應的特定文件頁面。不是首頁、也不是整套 docs — 是「相關的那一頁」。

**Source 權威性階層（依序）：**

| Priority | Source | Example |
|----------|--------|---------|
| 1 | Official documentation | react.dev、docs.djangoproject.com、symfony.com/doc |
| 2 | Official blog / changelog | react.dev/blog、nextjs.org/blog |
| 3 | Web standards references | MDN、web.dev、html.spec.whatwg.org |
| 4 | Browser/runtime compatibility | caniuse.com、node.green |

**非權威 — 永遠不要當作主要 source 來引用：**

- Stack Overflow 答案
- 部落格文章或教學（即使再有名）
- AI 生成的文件或摘要
- 你自己的 training data（這正是重點 — 要 verify）

**抓取要精準：**

```
BAD:  Fetch the React homepage
GOOD: Fetch react.dev/reference/react/useActionState

BAD:  Search "django authentication best practices"
GOOD: Fetch docs.djangoproject.com/en/6.0/topics/auth/
```

抓完之後，萃取出關鍵的 pattern，並記下任何 deprecation 警告或遷移指引。

當不同官方 source 互相衝突時（例如 migration guide 與 API reference 講法不同），把這個矛盾告訴使用者，並以偵測到的版本實際驗證哪個 pattern 真的可以運作。

### Step 3: 依照已記錄的 pattern 實作

寫出符合 documentation 內容的程式碼：

- 使用 docs 給的 API signature，不要憑記憶
- docs 若呈現新做法就用新做法
- docs 若 deprecate 某個 pattern，就不要再用舊版本
- docs 沒涵蓋的，要明確標示為未驗證

**當 docs 與既有專案 code 衝突時：**

```
CONFLICT DETECTED:
The existing codebase uses useState for form loading state,
but React 19 docs recommend useActionState for this pattern.
(Source: react.dev/reference/react/useActionState)

Options:
A) Use the modern pattern (useActionState) — consistent with current docs
B) Match existing code (useState) — consistent with codebase
→ Which approach do you prefer?
```

把這個衝突點出來。不要默默挑一個。

### Step 4: Cite Your Sources

每個 framework-specific pattern 都要附 citation。使用者必須能驗證每個決策。

**在程式碼註解中：**

```typescript
// React 19 form handling with useActionState
// Source: https://react.dev/reference/react/useActionState#usage
const [state, formAction, isPending] = useActionState(submitOrder, initialState);
```

**在對話中：**

```
I'm using useActionState instead of manual useState for the
form submission state. React 19 replaced the manual
isPending/setIsPending pattern with this hook.

Source: https://react.dev/blog/2024/12/05/react-19#actions
"useTransition now supports async functions [...] to handle
pending states automatically"
```

**Citation 規則：**

- 完整 URL，不要縮網址
- 盡量使用含 anchor 的 deep link（例如 `/useActionState#usage` 優於 `/useActionState`） — anchor 比起頂層頁面更能在文件改版後存活
- 在支撐非顯而易見決策時，引用對應段落
- 推薦 platform 功能時，附上 browser/runtime 支援資料
- 如果你找不到該 pattern 的官方文件，明確說出來：

```
UNVERIFIED: I could not find official documentation for this
pattern. This is based on training data and may be outdated.
Verify before using in production.
```

對「沒能驗證的部分」誠實，比假裝有把握更有價值。

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「我對這個 API 很有把握」 | 有把握不等於有證據。Training data 含有看起來正確、但對最新版本會壞掉的過時 pattern。要 verify。 |
| 「抓 docs 浪費 token」 | 幻想出一個 API 浪費更多。使用者除錯一小時，最後發現 function signature 變過了。一次 fetch 可避免好幾小時的重做。 |
| 「docs 不會有我要的東西」 | 如果 docs 沒涵蓋，這本身就是有用資訊 — 那個 pattern 可能根本不是官方推薦做法。 |
| 「我順手提一句『可能過時』」 | 一個 disclaimer 沒幫助。要嘛 verify 並 cite，要嘛清楚標示為 unverified。模糊兩可是最差選項。 |
| 「這只是小事，不用查」 | 用了錯誤 pattern 的小事會被當作 template。使用者會把你那個過時的 form handler 複製到十個元件裡，最後才發現有現代做法。 |

## Red Flags

- 寫 framework-specific code 卻沒有查該版本的 docs
- 用「我相信」或「我覺得」談 API，而不是引用 source
- 實作某個 pattern 卻不知道它適用於哪個版本
- 引用 Stack Overflow 或部落格而非 official documentation
- 因為 training data 中出現就用了 deprecated API
- 實作前沒讀 `package.json` / dependency 檔
- 交付的 code 對 framework-specific 決策沒有 source citation
- 只需要某一頁時卻 fetch 整個 docs 站台

## Verification

以 source-driven development 完成實作後：

- [ ] framework 與 library 版本已從 dependency 檔辨識出來
- [ ] 已抓取 framework-specific pattern 的 official documentation
- [ ] 所有 source 都是 official documentation，不是部落格或 training data
- [ ] code 遵循當前版本 documentation 中所示的 pattern
- [ ] 非平凡決策都附了完整 URL 的 source citation
- [ ] 沒有使用 deprecated API（已對照 migration guide）
- [ ] docs 與既有 code 之間的衝突已告知使用者
- [ ] 任何無法驗證的部分都被明確標示為 unverified
