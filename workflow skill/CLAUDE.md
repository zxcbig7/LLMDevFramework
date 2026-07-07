# Workflow Skill — milp-dev orchestrator

<system_context>
MILP 建模開發的工作流調度層。規範本體在 `../MILP Model/`（天條 + 三 phase 細則），
本資料夾只放 orchestrator skill：三階段 phase gate 推進 + status.json resume。
部署後由 Router 依「幫我建模 / 新題目 / tuning」等情境觸發。
</system_context>

<critical_notes>
- MUST skill 只做調度與 gate 把關，規則一律讀 `../MILP Model/` 對應 CLAUDE.md —— Why: 規範單一來源，skill 內複製規則必 drift
- MUST 改工作流只改本資料夾 `SKILL.md`，改建模規則去改 `../MILP Model/`
- 改動後 → 重新部署（對 Claude 說「讀 BOOTSTRAP.md，完成適配」）+ 跑 `/harness-eval milp-model`
</critical_notes>

<file_map>
SKILL.md - milp-dev skill 本體（部署到 ~/.claude/skills/milp-dev/）
../MILP Model/ - 規範單一來源（天條 + Model Design / Foundation Coding / Foundation Tuning）
</file_map>

<common_tasks>
- 手動啟動 → 對 Claude 說「幫我建模：<題目>」或「用 milp-dev 開新題目 <Project>」
- 續作 → 說「繼續 <Project>」（skill 依 status.json resume）
- 部署 → 已登錄 `bootstrap/inventory.json`（skill-milp-dev），BOOTSTRAP 適配即裝
</common_tasks>
