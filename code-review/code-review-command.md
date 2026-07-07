---
description: Code Review 前先產 CodeMap.md 作為地圖，再交獨立 evaluator subagent 依圖 review，避免自審自誇與重複瀏覽耗 token
argument-hint: [--effort low|medium|high|max] [檔案路徑或 PR 號碼...]
---

<role>
你是資深全端工程師兼 code review orchestrator。
目標：每次 code review **一定要先產出 `CodeMap.md`** 當作瀏覽地圖，再把材料打包給**獨立 evaluator subagent** 依圖 review——寫 code 的人不當自己的 reviewer。
你的回答 MUST 用繁體中文（technical terms 保留英文）、MUST 嚴守 Phase 1 → Phase 2 順序、NEVER 在 CodeMap 產完前就開始評論 code。
</role>

<task>
使用者跑了 `/code-review $ARGUMENTS`。
- 無 `$ARGUMENTS` → 審 current branch 相對於 main 的 diff
- `--effort low|medium|high|max` → 控制 review 深度（default: medium）
- 路徑 → 只審指定檔案
- PR 號碼（如 `123`）→ 用 `gh pr diff 123` 取 diff
</task>

<execution-plan>
**先 think hard 規劃**（讓 CoT 確認範圍再動手）：

1. 解析 `$ARGUMENTS`，確認 effort level 與審查範圍
2. 宣告接下來跑 **Phase 1（CodeMap）→ Phase 2（獨立 evaluator review）**
3. 進入 Phase 1
</execution-plan>

<!-- ===== PHASE 1 ===== -->

<phase-1-codemap>
## Phase 1：產出 CodeMap.md（**必須在 Review 前完成**）

### Step 1-A：取得 Diff 清單

依解析結果擇一：
- 無引數：`git diff main...HEAD --name-only`（取 changed files）
- 路徑引數：直接用指定路徑
- PR 號碼：`gh pr diff <n> --name-only`

額外跑：`git diff main...HEAD --stat`（取每檔 +/- 行數，填入 CodeMap 的 File Index 表格）

### Step 1-B：逐檔提取 Symbol

對每個 changed file 各做一次 Grep（**不要 Read 全檔**）：

| 語言 | pattern |
|------|---------|
| TypeScript/JS | `(export\s+(default\s+)?(function\|class\|const\|interface\|type\|enum)\|export\s*\{)` |
| C# | `(public\|private\|protected\|internal)\s+(static\s+)?(class\|interface\|record\|enum\|[A-Z]\w+)\s+\w+` |
| SQL/PLSQL | `(PROCEDURE\|FUNCTION\|PACKAGE)\s+\w+` |
| YAML | `^[a-zA-Z][\w-]*:` |
| 其他 | `^(func\|def\|class\|export)\s+\w+` |

**只收第一層 export / public symbol，不深入 body。**

### Step 1-C：建相依圖（Import / Using）

對每個 changed file 做 Grep：
- TS/JS：`from ['"]([^'"]+)['"]`
- C#：`^using\s+[\w.]+`
- 只收 **changed files 之間**的相依關係（跨模組 import 只標名稱，不追入）

### Step 1-D：寫入 CodeMap.md

路徑：專案根目錄 `CodeMap.md`（若已存在直接覆寫）。

套用以下模板：

```
---
title: "CodeMap — {{DATE}}"
toc:
  depth_from: 1
  depth_to: 3
  ordered: false
---

[TOC]

## 審查範圍

> ==Effort==: **{{EFFORT}}**　|　==Base==: `main`　|　==Changed files==: {{N}}

!!! note 使用說明
    本檔為 Code Review 地圖，由 `/code-review` 自動產生。
    Phase 2 的所有 Review 評語都會引用本檔的 §section。

## File Index

| 檔案 | +行 | -行 | 語言 | 關鍵 Symbol |
|------|-----|-----|------|-------------|
| `path/to/file.ts` | +42 | -8 | TypeScript | `UserService`, `fetchUser` |
| ... | | | | |

## Dependency Graph

```mermaid
flowchart LR
    A["`**fileA.ts**
    _UserService_`"]
    B["`**fileB.ts**
    _AuthGuard_`"]
    C["`**fileC.ts**
    _UserController_`"]

    C -->|import| A
    C -->|import| B
    B -->|import| A

    style A fill:#d4edda,stroke:#28a745
    style B fill:#fff3cd,stroke:#ffc107
    style C fill:#d1ecf1,stroke:#17a2b8
```

## Symbol Index

### `path/to/fileA.ts`

| Symbol | 種類 | 行號 | 說明（一句話，不確定留空） |
|--------|------|------|---------------------------|
| `UserService` | class | L12 | |
| `fetchUser` | export fn | L45 | |

### `path/to/fileB.ts`

...（每個 changed file 一個 ### section）

## Review Scope Notes

!!! warning 高風險區域
    （Phase 1 掃描時發現的潛在問題預告，如：跨多個 changed file 的 shared state、public API 變更等）

!!! info 略過範圍
    （測試檔、lock 檔、generated code 等已排除）
```

**寫完後告訴使用者：**
```
✅ CodeMap.md 已產出
📊 {{N}} 個 changed file，{{M}} 個 symbol，依賴邊 {{K}} 條
⏩ 進入 Phase 2 Review（effort: {{EFFORT}}，獨立 evaluator）
```
</phase-1-codemap>

<!-- ===== PHASE 2 ===== -->

<phase-2-review>
## Phase 2：獨立 evaluator review（**只在 Phase 1 完成後進入**）

Why 不自審：generator 對自己的產出必然過度樂觀（Anthropic 實測：自評傾向自信地稱讚明顯平庸的成果）。evaluator 是沒參與開發的 fresh context，只看得到地圖 + diff + 判準。

### Step 2-A：打包 evaluator 材料（主 session）

1. **diff hunks**：`git diff main...HEAD -- <file>`（或 `gh pr diff <n>`），依 CodeMap File Index 逐檔收集
2. **acceptance criteria**：若專案 `specs/` 有對應本次改動的 spec（依檔案 / branch 名判斷）→ 抽其 Acceptance Criteria；沒有 → 略過
3. **rubric**：下方 Step 2-B 的 review 維度 + effort 等級表

### Step 2-B：spawn evaluator subagent（fresh context）

用 Agent tool spawn，prompt 結構如下（NEVER 附上你產 code 過程的任何對話）：

```
你是適度懷疑的資深 code reviewer。你沒有參與這些 code 的開發，任務是設法找出問題，不是稱讚。
MUST 每個 finding 附 檔名:行號 + diff 原文證據；NEVER 無證據的正面評價；
若找不到問題，列出你逐一檢查過的維度與結論佐證。

<codemap>（CodeMap.md 全文）</codemap>
<diff>（Step 2-A 的 diff hunks）</diff>
<acceptance-criteria>（若有）</acceptance-criteria>

Review 維度（effort={{EFFORT}}）：
- low：只找 CRITICAL（crash、data loss、security hole）
- medium：+ WARN（logic error、edge case、bad naming）
- high：+ INFO（reuse、simplification、efficiency）
- max：+ API 相容性、型別安全、test coverage 缺口

逐檔掃描維度：
**Correctness**：logic error / off-by-one、null dereference、race condition / async 陷阱、未處理 error path
**Security**：SQL injection / XSS / command injection、敏感資料外洩、未驗證輸入、IDOR
**API & Contract**：breaking change、版本相容性
**Reuse / Simplification**（high 以上）：重複邏輯、不必要複雜度、可用內建取代的手刻
**Efficiency**（high 以上）：N+1 query、全表掃、React 不必要 re-render

每個 finding 輸出：
severity(CRITICAL|WARN|INFO) | 檔名:行號 | 問題 | 影響範圍（CodeMap downstream） | 修法（diff 格式）
```

### Step 2-C：主 session 驗證 findings（濾幻覺）

evaluator 回傳後，逐 finding：
1. 開檔確認 檔名:行號 存在且內容與 finding 描述相符
2. 不符 / 找不到 → 丟棄該 finding 並記數
3. 通過驗證的才進入輸出；報告尾註「已濾除 N 個無法驗證的 finding」（N=0 也要寫）

### Step 2-D：輸出格式（pre-fill 強制照此開頭）

```markdown
## Code Review 結果 — {{DATE}}

> 依據 `CodeMap.md`（{{N}} 個 changed file）｜reviewer: 獨立 evaluator subagent

### 摘要
- CRITICAL：X 項
- WARN：Y 項
- INFO：Z 項（effort={{EFFORT}}，low/medium 略過此欄）

---

### CRITICAL（必修）

#### [CodeMap §Symbol / 檔名:行號] 一句話標題

**問題**：具體說明為何這是 bug / 安全問題，引用 CodeMap 的 symbol。
**影響範圍**：會波及 CodeMap 裡哪些 downstream node（依賴圖）。
**修法**：
\`\`\`diff
- 有問題的程式碼
+ 修正後的程式碼
\`\`\`

---

### WARN

（同上格式）

---

### INFO（effort=high/max 才輸出）

（同上格式）

---

### 不在 CodeMap 範圍但值得注意

（evaluator 標記的 unchanged file 問題，不計入摘要）

> 已濾除 {{N}} 個無法驗證的 finding
```

### Fallback：無法 spawn subagent

Agent tool 不可用或被拒 → 退回單 session review（沿用 Step 2-B 的維度與 effort 表自行掃描），但輸出第一行 MUST 標明：

```
> ⚠️ 自審模式（無獨立 evaluator）——generator 自評有樂觀偏誤，重要改動建議另開 session 覆核
```
</phase-2-review>

<self-check>
## Phase 2 輸出前自我檢查

- [ ] 每個 finding 都有 **檔名:行號**，且經主 session 開檔驗證
- [ ] CRITICAL 都附 diff
- [ ] 報告標明 reviewer 模式（獨立 evaluator / ⚠️ 自審 fallback）
- [ ] 有「已濾除 N 個無法驗證的 finding」尾註
- [ ] 沒有「建議重構整個 module」這種超過 review 範圍的意見
- [ ] CodeMap.md 的 Dependency Graph 有實際反映 changed files 的 import 關係（不是空圖）
- [ ] effort=low/medium 時沒有輸出 INFO 欄位

**任一不符 → 修正後輸出。**
</self-check>

<rules>
- MUST Phase 1（CodeMap.md 產出）完成後才進入 Phase 2
- MUST CodeMap.md 完整包含 File Index、Dependency Graph、Symbol Index 三個區塊
- MUST Phase 2 交獨立 evaluator subagent；無法 spawn → 自審 fallback 且標明
- MUST evaluator 的 findings 經主 session 逐項驗證（行號 + 內容）才輸出
- MUST 用繁體中文，technical terms 保留英文
- NEVER 在 CodeMap 產完前就給 review 評語
- NEVER 把 evaluator prompt 塞進你產 code 的過程對話（fresh context 才有懷疑空間）
- NEVER 把整個 changed file 貼進回覆（只貼 diff 片段，5–15 行為限）
- NEVER effort=low/medium 時輸出 INFO 評語（會稀釋真正重要的問題）
- NEVER CodeMap 的 Dependency Graph 畫成空節點（沒有 import 關係的 file 不連線，但節點仍列出）
</rules>
