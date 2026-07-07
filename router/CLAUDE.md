# Router / Dispatch — 自適應分派層

<system_context>
框架最上層：讓 Claude 在**任何專案**自動判斷該套哪個 domain / skill，降低使用者手動指定。
兩部分：① `global-claude-block.md` 注入 `~/.claude/CLAUDE.md`（語言→domain + 任務→skill + 行為規則）
② `/scaffold` 半自動把 reference 框架的 CLAUDE.md 寫進專案。
規格：`specs/2026-06-06-framework-expansion-backlog.md` Item 0。
</system_context>

<critical_notes>
- MUST 改 dispatch 規則只改 `global-claude-block.md`（single source），再重新注入 `~/.claude/CLAUDE.md` —— Why: 兩處各改會 drift
- MUST 注入用 `<!-- LLMDEVFRAMEWORK:ROUTER START/END -->` 包夾 —— Why: idempotent，重裝只取代區塊內、不碰使用者其他全域設定
- NEVER scaffold 覆蓋既有 `CLAUDE.md` —— ALWAYS 偵測到就跳過並回報 —— Why: 會洗掉使用者既有設定
- NEVER 把 domain 規範正文複製進專案 —— ALWAYS 用 path reference —— Why: 框架一改、複製版就過時
</critical_notes>

<file_map>
router/CLAUDE.md                    - 本檔
router/global-claude-block.md       - 注入 ~/.claude/CLAUDE.md 的 Router 區塊（single source）
router/scaffold-command.md          - /scaffold slash command source
router/project-claude-template.md   - 專案 root CLAUDE.md 模板
router/subfolder-claude-template.md - 子資料夾 CLAUDE.md 模板
</file_map>

<paved_path>
**安裝 / 更新 Router 區塊**
1. 編輯 `global-claude-block.md`
2. 把 `{{FRAMEWORK_PATH}}` 換成框架實際路徑（部署位置，預設 `~/.claude/llmdevframework`）
3. 用 START/END marker 取代 `~/.claude/CLAUDE.md` 內舊區塊；沒有就 append

**裝 `/scaffold`**：`scaffold-command.md` → `~/.claude/commands/scaffold.md`，替換 `{{FRAMEWORK_PATH}}`
（以上由 root `BOOTSTRAP.md` 的 AI 適配流程執行，不依賴安裝腳本、不受 execution policy 影響）
</paved_path>

<hatch>
- 不想自動套 → 該專案 root `CLAUDE.md` 寫覆蓋規則（專案優先於全域）
- Router 是軟性（prompt 層）；要強制 → 另上 hook（本框架刻意不用，避開 execution policy 鎖死）
</hatch>

<fatal_implications>
- NEVER 手改 `~/.claude/CLAUDE.md` 內 marker 區塊（會被下次注入覆蓋）—— 要改去改 `global-claude-block.md`
- NEVER 讓 scaffold 在使用者 repo 自動寫檔不先列清單確認
</fatal_implications>
