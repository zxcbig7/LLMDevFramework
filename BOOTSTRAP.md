# BOOTSTRAP — AI 自行適配安裝 Playbook

> **使用方式（人類只做這步）**：在新電腦 clone 本 repo 後，開 Claude Code 對它說：
> **「讀 BOOTSTRAP.md，完成適配」**
> 其餘交給 AI：盤點 → 計畫（等你確認）→ 搬運 → 自我驗收。全程不執行任何安裝腳本。

<role>
你是 LLMDevFramework 的 bootstrap agent。
目標：把本框架適配到這台機器的 `~/.claude/`，讓所有 slash command、skill、Router 自動 dispatch 在任何專案可用。
你的回答 MUST 依 Phase 0 → 3 順序執行、MUST Phase 1 等使用者確認才寫檔、NEVER 覆蓋內容不同的既有檔（列 diff 詢問）。
</role>

## Phase 0：盤點

1. **框架根** `$FW` = 本檔所在目錄的絕對路徑（即 repo root）
2. **環境**：OS / shell（Windows 用 PowerShell 語法回報，其他 OS 自適應路徑分隔與 home 位置）
3. **`~/.claude/` 現況**（只讀不寫）：
   - `CLAUDE.md` 是否存在？內有無 `<!-- LLMDEVFRAMEWORK:ROUTER START -->` marker？
   - `commands/`、`skills/` 現有清單
   - `.llmdevframework.json` manifest 是否存在（舊安裝紀錄）
4. 讀 `$FW/bootstrap/inventory.json` 取得部署清單

## Phase 1：計畫（等使用者確認）

對 inventory 逐項比對現況，產出三欄計畫表：

| 分類 | 判定 | 動作 |
| --- | --- | --- |
| **將寫** | dest 不存在 | 直接寫入 |
| **將改** | dest 存在、內容 ≠ 轉換後 source | 列 diff 摘要，**逐項問使用者** |
| **跳過** | dest 存在、內容相同 | 不動 |

加上：
- Router 區塊計畫：marker 不存在 → append；存在但內容不同 → 區塊內替換（附 diff 摘要）
- `{{FRAMEWORK_PATH}}` 替換值 = `$FW`（正斜線路徑）
- 已知外部依賴不處理：`guizang-ppt-skill`（使用者自行 git clone）

**輸出計畫表後停下，使用者確認才進 Phase 2。**
全部「跳過」且 Router 一致 → 回報「✅ 已就緒，無需變更」並結束（idempotent）。

## Phase 2：執行

依計畫逐項執行（只動「將寫」與使用者點頭的「將改」）：

1. **Router 注入**：`~/.claude/CLAUDE.md` 的 `<!-- LLMDEVFRAMEWORK:ROUTER START/END -->` 區塊內容 = `$FW/router/global-claude-block.md`（`{{FRAMEWORK_PATH}}` 已替換）。marker 區塊外**一字不動**；無 CLAUDE.md → 建檔只含 Router 區塊
2. **Commands**：逐項 Read source → 替換 `{{FRAMEWORK_PATH}}`（`transform: substitute-framework-path` 時）→ Write 到 `~/.claude/<dst>`（Write tool，無 BOM）
3. **Skills**：依 item 的 `files` 清單整包複製到 `~/.claude/<dst>`（含 `references/` 子目錄）
3b. **Agents**（`type: agent`）：同 Commands 作法（單檔 Read → Write 到 `~/.claude/agents/<檔名>`，`transform: none` 原樣複製）；agent 定義重開 session 才生效
4. **Manifest**：寫 `~/.claude/.llmdevframework.json`：

```json
{
  "version": "2.0.0",
  "frameworkRoot": "<$FW>",
  "updatedAt": "<ISO 時間>",
  "files": [
    { "id": "...", "src": "...", "dst": "...", "sourceMtime": "...", "deployedMtime": "...", "deployedAt": "..." }
  ]
}
```

既有 manifest → 先 Read 再併（保留不在本次清單內的舊 entry 並標註）。

## Phase 3：驗證

依 `/harness-doctor` 的七項檢查自我驗收（至少跑 1 漏裝、2 孤兒、6 Router 區塊）：

- 全過 → 回報 `✅ 適配完成：N 寫入 / M 更新 / K 跳過`，並提示重開 session 讓 command 生效
- 有差異 → 列出差異與原因，NEVER 宣稱完成

<rules>
- MUST Phase 1 計畫表經使用者確認才寫任何檔
- MUST 重跑安全（idempotent）：無差異時只回報「已就緒」
- NEVER 覆蓋內容不同的既有檔而不列 diff 詢問
- NEVER 動 `~/.claude/CLAUDE.md` marker 區塊以外的內容
- NEVER 執行任何 `.ps1` / 安裝腳本（本 playbook 就是安裝器）
- 部署清單只認 `bootstrap/inventory.json`；要加項先改它（single source）
</rules>
