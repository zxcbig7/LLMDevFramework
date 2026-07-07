# Bootstrap — AI 自行適配安裝

<system_context>
框架的可攜層：新電腦 clone repo 後，對 Claude 說「讀 BOOTSTRAP.md 完成適配」，
AI 自行盤點 → 計畫 → 搬運 → 驗證，全程不依賴安裝腳本（零 .ps1、零 hook、零 execution policy 依賴）。
取代已廢棄的 scripts/ 安裝路線（deploy.config.json 內容已遷移為本目錄 `inventory.json`）。
規格：`specs/2026-07-03-harness-iteration-loop.md` Part 2 + Part 4。
</system_context>

<critical_notes>
- MUST 改部署清單只改 `inventory.json`（single source）—— Why: 清單存在兩處必 drift
- MUST BOOTSTRAP Phase 1 列「將寫 / 將改 / 跳過」清單、使用者確認才執行 —— Why: 在使用者機器寫檔不先列清單是 scaffold 同款禁令
- NEVER 覆蓋內容不同的既有檔 —— ALWAYS 列 diff 詢問 —— Why: 會洗掉使用者本機調整
- MUST 重跑安全（idempotent）：全部「跳過」且 Router 一致 → 只回報「已就緒」不寫檔
- manifest 沿用 `~/.claude/.llmdevframework.json` v2 schema（frameworkRoot + files[]），既有的先 Read 再併
</critical_notes>

<file_map>
bootstrap/CLAUDE.md                 - 本檔（domain 規範）
bootstrap/inventory.json            - 部署清單 single source（command + skill）
bootstrap/harness-doctor-command.md - `/harness-doctor` slash command source（七項對帳）
../BOOTSTRAP.md                     - repo root 的 AI 適配 playbook（入口）
</file_map>

<paved_path>
- 新機器適配 → clone → 對 Claude 說「讀 BOOTSTRAP.md 完成適配」
- 加部署項 → 改 `inventory.json` → 重跑 BOOTSTRAP（Phase 1–3）
- 健檢 → `/harness-doctor`
- doctor 七項：漏裝 / 孤兒 / stale source / 使用者改過 dest / CodeMap 行數 / Router 區塊 / eval case STALE（細節見 `harness-doctor-command.md`）
</paved_path>

<fatal_implications>
- NEVER 動 `~/.claude/CLAUDE.md` marker 區塊以外的內容
- NEVER doctor 修任何檔（read-only，只報告）
- NEVER 把 guizang-ppt-skill 納入 inventory（外部 git clone 依賴，來源記錄於 README）
</fatal_implications>
