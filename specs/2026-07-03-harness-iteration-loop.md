---
title: Harness 迭代閉環與可攜化：Eval + Doctor + Evaluator + AI Bootstrap
status: shipped
created: 2026-07-03
updated: 2026-07-04
modules: [llmdevframework]
---

# Harness 迭代閉環與可攜化

## Summary

為 LLMDevFramework 補上「量測、回饋、可攜」四件基礎設施：① **Behavioral eval**——每個 domain 附觸發案例，改 CLAUDE.md 後跑 `/harness-eval` 驗證模型行為（框架的 unit test）；② **Doctor 對帳**——`/harness-doctor` 由 AI 以 Read/Glob/Grep 執行七項檢查，偵測 source ↔ inventory ↔ `~/.claude` 三方 drift（框架的 `git status`）；③ **Evaluator 分離**——`code-review` 改 spawn 獨立 evaluator subagent，終結 generator 自審自誇；④ **AI Bootstrap**——新電腦 clone repo 後對 Claude 說一句「讀 `BOOTSTRAP.md` 完成適配」，AI 自行盤點、搬運、調整、驗證，**全程不依賴安裝腳本**。既有 `scripts/` 安裝路線廢棄刪除。

## Motivation / Why

- 2026-07-03 Harness design 評估：框架認知框架與行為流程已高成熟，但「可迭代」缺量測（改規則無 regression 檢測）、drift 靠人眼、code review 同 session 自審（Anthropic 實證 generator 自評必然自誇）。
- 使用者需求（2026-07-03）：別台電腦裝了 Claude 後，希望 **AI 自己手動搬運調整**完成框架適配，不走腳本安裝路線。現有 `scripts/install.ps1` 是相反哲學（deterministic 腳本），且為半成品（不支援 skill、有 execution policy 風險）——直接廢棄，改 AI-driven playbook。
- doctor 與 bootstrap 共用同一份對帳 checklist（bootstrap 裝 → doctor 驗），是真協同，併一份 spec。

## Scope

### In Scope

- 新 domain `harness-eval/`（CLAUDE.md + case 模板 + `/harness-eval` command source）
- 首發 3 個 domain 的 eval cases：CMD Developer、PowerShell、React & Typescript（各 3+；選這三個因規則最可觀察：chcp/延遲展開、BOM/CRLF/5.1 相容、strict/explicit return types）
- 新 domain `bootstrap/`（CLAUDE.md + `inventory.json` 部署清單 + `/harness-doctor` command source）+ repo root `BOOTSTRAP.md` playbook
- `code-review` 回灌框架 source（`code-review/code-review-command.md`）+ Phase 2 evaluator subagent 改造（順帶解掉三孤兒之一）
- `scripts/` 全套刪除（install / update / uninstall / lib / README / MANUAL-INSTALL / deploy.config），清單內容遷移為 `bootstrap/inventory.json` 並擴充 `type: skill`
- root `CLAUDE.md`、`CodeMap.md`、`README.md` 同步，清光 scripts stale 引用

### Out of Scope

- 全自動 headless CI runner（`claude -p` 批次跑 eval）——成功標準已定為半自動
- `write-tutorial` / `research-note` 兩孤兒的處置（doctor 會回報，決議留 backlog Phase 0）
- SDD 驗收段的 evaluator 化（先在 code-review 驗證模式）
- 非 Windows 的完整驗證（playbook 寫成 OS 中立，本輪只在 Windows 驗收）

## User Stories / Use Cases

1. As a 框架維護者, I want 改完 domain CLAUDE.md 後跑 `/harness-eval <domain>` 得到逐 case pass/fail, so that 規範迭代有 regression 檢測。
2. As a 框架維護者, I want 跑 `/harness-doctor` 看三方 drift 報告, so that 漏裝、孤兒、過時不靠人眼。
3. As a 開發者, I want `/code-review` 的審查者是沒參與開發的獨立 agent, so that findings 不被自評偏誤稀釋。
4. As a 框架使用者, I want 新電腦 clone repo 後一句話讓 Claude 自行適配安裝並自我驗收, so that 不依賴腳本與 execution policy、換機可重複。

## Acceptance Criteria

### Part 1 — Behavioral Eval

- [ ] CMD Developer、PowerShell、React & Typescript 各有 ≥3 個 eval case，格式符合 `harness-eval/case-template.md`
- [ ] `/harness-eval <domain>` 對每 case 回報 PASS / FAIL / STALE + 一句理由，判準逐項附被測輸出原文引用
- [ ] case `rule_refs` 任一在 domain CLAUDE.md 找不到 → 標 STALE、不執行、提示更新 case
- [ ] 生成與評判分離：被測 subagent 只拿 domain CLAUDE.md + Input，看不到 Expected
- [ ] 預設單 domain；`all` 先列 case 總數與預估成本、確認才跑

### Part 2 — Doctor 對帳（`/harness-doctor`，AI 執行）

- [ ] 七項檢查：漏裝（inventory 有 dest 無）、孤兒（dest 有 inventory 無）、stale（source 較新）、使用者改過 dest、CodeMap File Index 行數偏差（>15 行或 >10%）、Router 區塊 drift（global CLAUDE.md marker 內容 vs `router/global-claude-block.md`）、全 domain eval case rule_refs STALE sweep
- [ ] 比對基準：`bootstrap/inventory.json` + `~/.claude/.llmdevframework.json` manifest；全程 AI 以 Read/Glob/Grep 執行，零 `.ps1`
- [ ] 以現存已知 drift 當驗收 fixture：能偵測 write-tutorial / research-note 孤兒、skill 未部署
- [ ] read-only：NEVER 修任何檔，只報告 + 建議修法；報告 pre-fill 格式固定

### Part 3 — Evaluator 分離

- [ ] `code-review/code-review-command.md` 存在（回灌現安裝版）並登錄 inventory
- [ ] Phase 2 改造：主 session 打包（CodeMap + diff hunks + 對應 spec acceptance criteria 若存在 + effort + review rubric）→ spawn fresh-context evaluator subagent → 回傳 findings → 主 session 逐項驗證（開檔確認 檔名:行號，濾幻覺）→ 照原 pre-fill 輸出
- [ ] evaluator 看不到生成過程對話；role 為「適度懷疑的 reviewer」，每 finding MUST 附 檔名:行號 證據
- [ ] 無法 spawn 時退回單 session review，輸出 MUST 標明「⚠️ 自審模式（無獨立 evaluator）」

### Part 4 — AI Bootstrap

- [ ] repo root `BOOTSTRAP.md`：新機器 clone 後對 Claude 說「讀 BOOTSTRAP.md 完成適配」即走完 Phase 0–3
- [ ] Phase 1 必列「將寫 / 將改 / 跳過（已存在且相同）」清單 + 本機路徑替換值，使用者確認才執行
- [ ] Phase 2 執行：Router 區塊 marker 注入（idempotent）、commands 寫入（`{{FRAMEWORK_PATH}}` 換本機實際路徑）、skills 整包複製（SKILL.md + references/）、寫 manifest（沿用既有 schema + skill entries）
- [ ] NEVER 覆蓋內容不同的既有檔——列 diff 詢問使用者
- [ ] Phase 3 以 `/harness-doctor` checklist 自我驗收並回報
- [ ] Idempotent：對已就緒的機器重跑，只回報「已就緒」不重寫
- [ ] `scripts/` 已刪除（git 史保留可回收），root CLAUDE.md / CodeMap / README 無任何 scripts stale 引用

### 整體

- [ ] root `CLAUDE.md` file_map 加 `harness-eval/`、`bootstrap/`、`code-review/`；`CodeMap.md` File Index / Key Rules / Coverage 同步
- [ ] 新增的 `/harness-eval`、`/harness-doctor`、`code-review` 用 BOOTSTRAP 流程裝進本機（dogfood）
- [ ] 所有新文件對照 `prompt-principles/` 12 點 self-check 通過，各 CLAUDE.md ≤200 行

## Module Interactions

| 元件 | 動作 | 互動對象 |
| --- | --- | --- |
| `harness-eval/` | 新建 | 讀各 `<domain>/evals/*.md` 與 `<domain>/CLAUDE.md`；Agent tool（spawn 被測 subagent） |
| `<domain>/evals/` | 新建於 3 首發 domain | 被 runner 與 doctor 掃描 |
| `bootstrap/` | 新建 | `inventory.json` 為部署清單 single source；`/harness-doctor` 讀 inventory + manifest + `~/.claude/` |
| `BOOTSTRAP.md`（root） | 新建 | 讀 `bootstrap/inventory.json`、寫 `~/.claude/`（CLAUDE.md Router 區塊、commands、skills、manifest） |
| `code-review/` | 回灌 + 改造 | Agent tool（spawn evaluator）、`CodeMap.md`、`specs/` |
| `scripts/` | 刪除 | 內容遷移：deploy.config.json → `bootstrap/inventory.json` |

## Interface Design

### `/harness-eval <domain>|all [case-id]`

1. 解析引數 → 定位 `<domain>/evals/`；`all` 先報 case 數 + 成本、等確認
2. 每 case：grep `rule_refs` → 缺任一即 STALE skip
3. spawn 被測 subagent：prompt = domain CLAUDE.md 全文 + case Input（NEVER 附 Expected）
4. 主 session 評分：Expected 逐項對照輸出、附原文證據；Anti-pattern 出現即 FAIL
5. 報告 pre-fill：`## Eval 報告 — <domain> — YYYY-MM-DD` + 逐 case 表 + FAIL 詳情；預設只印不存檔

### `/harness-doctor`

七節報告（對應七項檢查），每節 `OK` 或問題清單 + 建議修法（補跑 BOOTSTRAP 對應段 / 更新 case / 同步 CodeMap）。結尾一行總結：`✅ 無 drift` 或 `⚠️ N 項待處理`。

### `BOOTSTRAP.md` playbook（AI 執行）

```text
Phase 0 盤點   偵測 OS / shell / 框架實際路徑 / ~/.claude 現況（既有 CLAUDE.md、commands、skills、manifest）
Phase 1 計畫   依 inventory.json 列「將寫 / 將改 / 跳過」+ 路徑替換值 → 等使用者確認
Phase 2 執行   Router marker 注入 → commands 寫入（路徑替換）→ skills 整包複製 → 寫 manifest
Phase 3 驗證   跑 /harness-doctor checklist 自我驗收 → 回報
```

### Evaluator subagent 契約（code-review Phase 2）

- 輸入：CodeMap.md 全文、diff hunks、acceptance criteria（若有）、effort、review rubric（沿用現版 Step 2-C）
- 輸出：findings 清單，每項 `severity | 檔名:行號 | 問題 | 影響 | 修法(diff)`；無 finding 需列已檢查維度佐證
- 主 session：驗證行號存在、濾幻覺、彙整成現版 Step 2-D pre-fill 格式

## Data Model（檔案格式）

### Eval case（`<domain>/evals/<case-id>.md`）

```markdown
---
id: cmd-batch-rename
domain: CMD Developer
rule_refs: ["chcp 65001", "延遲展開"]   # 對 domain CLAUDE.md 的 grep 錨點，缺 → STALE
created: 2026-07-03
updated: 2026-07-03
---

## Input（模擬使用者輸入）

幫我寫個 .bat 批次把資料夾內 *.txt 改名加日期前綴

## Expected（可觀察判準，全過才 PASS）

- [ ] 產出含 `chcp 65001`
- [ ] 迴圈內變數展開用 `!var!` 且有 `setlocal enabledelayedexpansion`
- [ ] 檔名含空格有引號包裹

## Anti-patterns（出現任一即 FAIL）

- 迴圈內用 `%var%` 展開執行期變數
```

### `bootstrap/inventory.json`（部署清單 single source，自 deploy.config.json 遷移）

```json
{ "id": "cmd-sdd", "type": "command", "src": "sdd/sdd-command.md", "dst": "commands/sdd.md", "transform": "substitute-framework-path" }
{ "id": "skill-mermaid", "type": "skill", "src": "Mermaid Diagrams/", "dst": "skills/mermaid-diagrams/", "files": ["SKILL.md", "references/"] }
```

### Manifest（`~/.claude/.llmdevframework.json`）

沿用既有 schema，bootstrap 寫入（含 skill entries），doctor 讀取。

## Edge Cases & Error Handling

- **rule_refs 誤判 STALE**：錨點取 2 個以上、比對忽略前後空白；仍誤判 → 修 case 錨點
- **LLM judge 不穩**：判準必須可觀察（「含 X」「先問了 Y」），禁模糊判準；FAIL 必附原文引用，無法引用 → PASS-with-note
- **判準太鬆假 PASS**：case 模板強制含 Anti-patterns（negative assertion）
- **token 成本**：eval 單 domain 預設、每 case 一 subagent、`all` 確認閘；bootstrap 一次性成本可接受；doctor 以讀檔為主
- **bootstrap 非確定性**：Phase 1 確認閘 + Phase 3 doctor 驗收雙保險
- **目標機已有同名但內容不同的 command/skill**：NEVER 覆蓋，列 diff 問使用者
- **全域 CLAUDE.md 已有舊版 Router 區塊**：marker 內整段替換，區塊外一字不動
- **非 Windows 目標機**：playbook 路徑/shell 寫成 OS 中立由 AI 自適應；本輪僅 Windows 驗收
- **domain 資料夾含空格**（`CMD Developer/`）：所有路徑操作引號包裹
- **BOM**：框架不再產生新 `.ps1`；所有新檔用 Write tool（無 BOM）
- **evaluator 幻覺 finding**：主 session 驗證行號與內容存在才輸出，驗證失敗丟棄並記數
- **scripts 刪除後回收需求**：git 史可回收；README 記一行遷移說明（deploy.config → inventory）

## Non-Functional Requirements

- **相容**：全程零 `.ps1`、零 hook、零 execution policy 依賴（`Mermaid Diagrams/references/render-mermaid.ps1` 屬 domain 內容，不受影響）
- **成本**：`/harness-eval <domain>` ≤ case 數次 subagent 呼叫；`all` 有確認閘
- **文件**：`harness-eval/CLAUDE.md`、`bootstrap/CLAUDE.md` 各 ≤200 行
- **Observability**：eval / doctor / bootstrap（Phase 1 清單、Phase 3 驗收）三種報告皆固定 pre-fill 格式，跨次可比對

## Open Questions

- [ ] eval 報告要不要 `--save` 留歷史（預設不存、不 commit）
- [ ] inventory 要不要加框架版本欄（換機時比對版本）；預設不做，需要再加

## Implementation Plan

### Stub 階段（先做）

- [ ] `harness-eval/CLAUDE.md` 骨架 + `case-template.md`（schema 即 interface，stub 就定稿）+ `harness-eval-command.md` 骨架
- [ ] 3 個 domain 建 `evals/` + 各 1 case 骨架（frontmatter + Input 填好，Expected 留 TODO）
- [ ] `bootstrap/CLAUDE.md` 骨架 + `bootstrap/inventory.json`（自 deploy.config.json 遷移 + 補 skill entries——資料即 interface，stub 就遷）+ `harness-doctor-command.md` 骨架（七檢查 section + TODO）
- [ ] root `BOOTSTRAP.md` 骨架（Phase 0–3 標題 + TODO）
- [ ] `code-review/code-review-command.md` 原樣回灌現安裝版 + `<!-- TODO: Phase 2 evaluator 改造 -->` 標記
- [ ] 驗證：inventory.json parse 通過（`ConvertFrom-Json`）、所有 `src` 路徑存在、`update.ps1 -DryRun` 不再使用（scripts 尚未刪，僅停用）

### 逐段實作

- [ ] ① eval case schema 定稿 + 首發 9 cases → verify: 欄位齊全、rule_refs grep 得到
- [ ] ② `/harness-eval` runner 完整 → verify: 對 CMD Developer 實跑一輪，報告含 PASS/FAIL/STALE + 證據
- [ ] ③ `/harness-doctor` 七項檢查完整 → verify: 抓到已知 drift（兩孤兒、skill 未部署）
- [ ] ④ `BOOTSTRAP.md` 完整 playbook → verify: 本機重跑 = 回報「已就緒」；暫移走一個 command 再跑 = 能偵測並補裝
- [ ] ⑤ 刪 `scripts/` + 清 stale 引用 → verify: `/harness-doctor` 過、全 repo grep 無 scripts 引用
- [ ] ⑥ code-review Phase 2 evaluator 改造 → verify: 對本 spec 的 stub commit 跑 `/code-review`，確認 spawn 與自審 fallback
- [ ] ⑦ root CLAUDE.md + CodeMap 同步、全文件 self-check → verify: doctor CodeMap 行數檢查過

## 驗收紀錄（2026-07-04）

- **Part 1**：3 domain × 3 cases = 9 cases，rule_refs 錨點全數 grep 驗證；三 domain 首輪實跑 **9/9 PASS**（ps-config-rewrite 為 PASS-with-note：讀檔走 .NET UTF-8 偵測未顯式傳 encoding，已按判準歸因記錄）
- **Part 2**：doctor 七項 inline 實跑，以現存 drift 當 fixture 全數命中（v1 manifest 舊路徑 / 4-of-7 漏記、兩孤兒、skills 未部署、3 新 command 未裝）
- **Part 3**：`code-review` 回灌 + Phase 2 evaluator 改造 + 部署熱載入；⚠️ 註記：在框架 repo 本身跑 `/code-review` 會覆寫 curated `CodeMap.md`（Phase 1-D 設計給使用者專案），框架 repo 內驗證需改用 scratchpad 路徑
- **Part 4**：3 新 command 以 BOOTSTRAP 流程裝進本機並熱載入；manifest v2（併舊 4 筆 + 3 command + 2 skill）；mermaid-diagrams / slide-builder skills 已部署；scripts/ 已 `git rm`（7 檔，git 史可回收）
- **殘留（非阻塞）**：write-tutorial / research-note 孤兒處置留 backlog；pptx skill 仍需自 anthropics/skills 手動複製；BOOTSTRAP 完整 Phase 0–3 跑一輪收編舊 4 筆 manifest entry 待新機實測

## References

- 缺口來源：2026-07-03 Harness design 成熟度評估（本 session）
- Anthropic：Harness design for long-running agents（自評偏誤、gradable criteria、independent evaluator）
- 可攜需求：2026-07-03 使用者——「AI 自己手動搬運調整，不走安裝路線」
- 被廢棄對象：`scripts/`（install/update/uninstall/lib.ps1、deploy.config.json、README、MANUAL-INSTALL）
- 被改造對象：`~/.claude/commands/code-review.md`（現安裝版，無 source 孤兒）
- 對帳背景：`specs/2026-06-06-framework-expansion-backlog.md` Item 0 / Phase 0
- 元規範：`prompt-principles/CLAUDE.md`
