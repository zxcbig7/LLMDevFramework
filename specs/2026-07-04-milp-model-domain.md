---
title: MILP Model Domain — 數學模型開發 AI 輔助文件整合
status: approved-in-session
created: 2026-07-04
updated: 2026-07-04
modules: [MILP Model, workflow skill, router, bootstrap, harness-eval]
---

# MILP Model Domain + milp-dev Workflow Skill

## Summary

把 OptimizationFramework 的兩份既有 AI 輔助資產（`ClaudeAIAssistant/`、`AI Modeling - Claude Code/`）整合成 LLMDevFramework 的正式 domain：`MILP Model/`（三階段規範）+ `workflow skill/`（`milp-dev` orchestrator skill），並接上框架標準配套（Router dispatch、inventory 部署、harness-eval cases）。

## Motivation / Why（整合度不高的診斷）

兩份資產各自演化，衝突與重複並存：

| 面向 | ClaudeAIAssistant（較新，天條） | AI Modeling - Claude Code（較舊 / Web API 產品） |
| --- | --- | --- |
| 命名 | `Parameter_Xxx`、`VariableB_/X_/I_` | `Param_Xxx`、`VariableY_`（整數+二元混用） |
| 專案結構 | 五資料夾（Set/Parameter/Variable/Objective/Constraint） | 兩資料夾（Model/ + Project/），單檔多 class |
| Dataload | Sets 由 Parameters 衍生（天條） | Sets 手寫 `List<string>.AddRange` |
| 建模方式 | generator `[OptVar]`/`[OptParam]` + Fluent `OptModel` 預設 | 純手寫 |
| 工作流 | 三階段 phase gate（Modeling→Coding→Tuning） | 16-stage pipeline + stages/ 檔 + status.json |
| CLAUDE.md | 356 行 | 541 行（皆超框架 200 行上限，且大量重複 API 內容） |

痛點：Claude 進 OptimizationFramework 時沒有單一權威可循，兩份規範互相打架；LLMDevFramework 的 Router / eval / bootstrap 配套完全沒接上。

## Design Decisions

1. **單一真相**：`$FW/MILP Model/` 為 Claude Code 互動建模的 canonical 規範。重量級參考（developer-guide.md、CPLEX_API_REFERENCE.md、claudemdTemplate/、範例專案）留在 OptimizationFramework，一律 path reference（DRY）。
2. **命名 / 慣例基準**：ClaudeAIAssistant 天條全勝（較新、經 dogfood）。AI Modeling 的 `Param_`/`VariableY_`/兩資料夾結構定位為 Web API pipeline 專用 legacy，不進本 domain。
3. **16-stage pipeline 定位**：屬 `AI Modeling - Claude Code` 的 Web API 產品內部實作（PromptLibrary 消費），Claude Code 互動流程不採用；互動流程 = 三階段 + 直接產出 Model.md 與 .cs（保留輕量 status 供 resume）。
4. **三階段 = 資料夾結構**：`Model Design/`（Phase 1）、`Foundation Coding/`（Phase 2）、`Foundation Tuning/`（Phase 3）。修正使用者手建資料夾 typo（`Model Desigh`、`Foundation Turning` 為空資料夾，棄用）。
5. **可評測天條放 domain root**：`/harness-eval` 被測 subagent 只吃 domain root CLAUDE.md，故 phase gate / 命名 / LHS-RHS / hardcode / tuning gate 等硬規則必須寫在 `MILP Model/CLAUDE.md` 本體；子資料夾放各 phase 細節。
6. **workflow skill 形態**：skill（非 slash command），名 `milp-dev`，source 放 `workflow skill/`，部署到 `~/.claude/skills/milp-dev/`。觸發語：「幫我建模」「新題目」「MILP」「最佳化問題」等。

## Scope

### In Scope

- `MILP Model/CLAUDE.md` + 三個 phase 子 CLAUDE.md + `evals/` 3 cases
- `workflow skill/SKILL.md`（milp-dev）+ 薄 CLAUDE.md
- `router/global-claude-block.md` 加 dispatch 規則（情境 + `.cs`+OptimFoundation 偵測）
- `bootstrap/inventory.json` 登錄 skill-milp-dev
- root `CLAUDE.md` file_map、`CodeMap.md`、`README.md` 同步

### Out of Scope（後續另議）

- 動 OptimizationFramework 側任何檔案。migration plan：
  - `ClaudeAIAssistant/CLAUDE.md` 瘦身 → 保留 DLL 路徑 / 專案清單等 local 事實，規範正文改 reference `$FW/MILP Model/`
  - `AI Modeling - Claude Code/CLAUDE.md` 移除「Framework 1：Claude Code 互動建模工作流」段（改由 milp-dev skill 承擔），只留 Web API 產品規範
- 不動 OptimFoundation DLL / Prompts/*.md / Web API code

## Acceptance Criteria

- [ ] `MILP Model/CLAUDE.md` ≤200 行，含可 grep 的天條錨點（phase gate、命名、LHS/RHS、hardcode、tuning gate）
- [ ] 三個 phase 子 CLAUDE.md 各 ≤200 行，XML tag 結構，good/bad 對照至少各 1 組
- [ ] `workflow skill/SKILL.md` frontmatter 含 name + description（觸發語完整），流程含 phase gate 與 resume
- [ ] 3 個 eval case rule_refs 錨點全數 grep 得到 root CLAUDE.md
- [ ] `inventory.json` 合法 JSON、新增 skill-milp-dev
- [ ] Router 區塊新增 MILP dispatch；CodeMap / README / root CLAUDE.md 同步
- [ ] 全部新文件過 prompt-principles 12 點 self-check
