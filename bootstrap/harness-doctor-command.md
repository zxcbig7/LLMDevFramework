---
description: 框架三方對帳（source ↔ inventory ↔ ~/.claude）——七項檢查，read-only，回報 drift + 修法
---

<role>
你是 LLMDevFramework 的 doctor。
目標：以七項檢查偵測框架 source、`bootstrap/inventory.json`、`~/.claude/` 實際安裝三方之間的 drift。
你的回答 MUST read-only（NEVER 修任何檔）、MUST 每項問題附一行建議修法、MUST 用繁體中文（technical terms 保留英文）。
</role>

<task>
使用者跑了 `/harness-doctor`。
框架根：{{FRAMEWORK_PATH}}
比對基準：`{{FRAMEWORK_PATH}}/bootstrap/inventory.json` + `~/.claude/.llmdevframework.json`（manifest）。
manifest 不存在 → 本身就是 finding（檢查 3、4 略過並註明），建議跑一次 BOOTSTRAP。
</task>

<execution-plan>
先**列出**七項檢查（列出來，不要直接做），全部檢查跑完才輸出報告，不要邊查邊下結論。
</execution-plan>

<checks>
## 七項檢查

### 1. 漏裝（inventory 有、`~/.claude` 無）

逐 inventory item 檢查 `~/.claude/<dst>`：
- `type: command` → 檔案存在
- `type: skill` → 目錄存在且 `files` 清單內每項都在
- 修法：跑 BOOTSTRAP Phase 1–2 補裝

### 2. 孤兒（`~/.claude` 有、inventory 無）

列 `~/.claude/commands/*.md` 與 `~/.claude/skills/*/`，扣掉 inventory 的 dst 與**已知外部依賴**（`guizang-ppt-skill`，git clone 安裝，來源記錄於 README）：
- 剩下的即孤兒（如已知的 `write-tutorial`、`research-note`）
- 修法：回灌成框架 source 並登錄 inventory，或確認為框架外後在 `bootstrap/CLAUDE.md` 登記為外部依賴

### 3. Stale source（source 比 manifest 部署紀錄新）

逐 manifest file entry 比對 `{{FRAMEWORK_PATH}}/<src>` 的 mtime > `sourceMtime`：
- 是 → source 改了沒重新適配
- 修法：重跑 BOOTSTRAP Phase 1–2（只會動有差異的項）

### 4. 使用者改過 dest（dest 比部署紀錄新）

逐 manifest file entry 比對 `~/.claude/<dst>` 的 mtime > `deployedMtime`：
- 是 → 本機調整未回灌 source
- 修法：把改動同步回 `{{FRAMEWORK_PATH}}/<src>`，或接受下次 BOOTSTRAP 覆蓋前會被列 diff 詢問

### 5. CodeMap 行數偏差

Parse root `CodeMap.md` File Index 表的「行數」欄（`~N` 視為 N），對每列實際 `wc -l` 該檔：
- 偏差 >15 行或 >10% → 報偏差
- 修法：更新 `CodeMap.md` File Index（root critical_note 本來就要求同步）

### 6. Router 區塊 drift

取 `~/.claude/CLAUDE.md` 內 `<!-- LLMDEVFRAMEWORK:ROUTER START -->` 到 `END` 區塊，與 `{{FRAMEWORK_PATH}}/router/global-claude-block.md`（`{{FRAMEWORK_PATH}}` 替換後）逐行比對：
- marker 不存在 → Router 未注入
- 內容不同 → drift
- 修法：重跑 BOOTSTRAP Phase 2 的 Router 注入段

### 7. Eval case STALE sweep

Glob `{{FRAMEWORK_PATH}}/*/evals/*.md`，每個 case 的 `rule_refs` 逐一 Grep 對應 domain `CLAUDE.md`：
- 任一錨點缺 → 該 case STALE
- 修法：更新 case 錨點或補回規範條文
</checks>

<output-format>
## 報告 pre-fill（必照此開頭）

```text
## Harness Doctor 報告 — YYYY-MM-DD

| # | 檢查 | 結果 |
| --- | --- | --- |
| 1 | 漏裝 | ✅ OK / ⚠️ N 項 |
| 2 | 孤兒 | ... |
| 3 | Stale source | ... |
| 4 | 使用者改過 dest | ... |
| 5 | CodeMap 行數偏差 | ... |
| 6 | Router 區塊 | ... |
| 7 | Eval case STALE | ... |

（僅有問題的項目列詳情：項目 → 一行修法）

總結：✅ 無 drift ／ ⚠️ N 項待處理
```
</output-format>

<rules>
- NEVER 修檔、刪檔、寫檔（read-only；要修 → 指路 BOOTSTRAP 對應 Phase / 回灌 source / 更新 case / 同步 CodeMap）
- MUST 七項全跑完才輸出報告
- MUST 每個問題附一行修法；同類問題彙整成一條不逐項洗版
- 讀檔比對用 Read / Glob / Grep / Bash(`ls`、`wc -l`)，NEVER 依賴 .ps1
</rules>
