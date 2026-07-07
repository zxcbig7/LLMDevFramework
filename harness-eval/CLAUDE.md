# Harness Eval — 框架行為評測

<system_context>
框架的 unit test 層。每個 domain 附 eval cases（模擬輸入 + 可觀察判準 + anti-patterns），
改 domain CLAUDE.md 後跑 `/harness-eval <domain>` 驗證模型行為是否仍符合規範。
生成與評判分離：被測 subagent 只拿 domain CLAUDE.md + Input，NEVER 看到 Expected。
規格：`specs/2026-07-03-harness-iteration-loop.md` Part 1。
</system_context>

<critical_notes>
- MUST 判準寫成可觀察行為（「產出含 X」「先問了 Y」），NEVER 用「品質好」「合理」類模糊字
  Why: LLM judge 只有在能引用原文證據時才穩定可重複
- MUST 每個 case 含 Anti-patterns 區（出現任一即 FAIL）
  Why: 只有正向判準會假 PASS，negative assertion 才抓得到「做了不該做的事」
- MUST case `rule_refs` 是 domain CLAUDE.md 的 grep 錨點（≥2 個）；任一找不到 → 該 case 標 STALE 不執行
  Why: 規範改了 case 沒跟上，跑出來是假訊號
- MUST `all` 先報 case 總數與預估成本（1 case ≈ 1 subagent 呼叫）、使用者確認才跑
  Why: eval 是 token 密集操作，成本要先可見
</critical_notes>

<file_map>
harness-eval/CLAUDE.md               - 本檔（domain 規範）
harness-eval/case-template.md        - eval case 模板（schema 已定稿）
harness-eval/harness-eval-command.md - `/harness-eval` slash command source
../<domain>/evals/*.md               - 各 domain 的 cases（首發：CMD Developer、PowerShell、React & Typescript）
</file_map>

<paved_path>
- 跑一輪 eval → 改完該 domain CLAUDE.md 後 `/harness-eval <domain>`
- 新增 case → 複製 `case-template.md` 到 `<domain>/evals/<case-id>.md`；rule_refs 寫入前先 grep 驗證錨點存在
- runner 工作流：收集 cases → rule_refs STALE 檢查 → 逐 case spawn 被測 subagent → 主 session 逐判準評分（引用原文）→ pre-fill 報告（細節見 `harness-eval-command.md`）
</paved_path>

<fatal_implications>
- NEVER 把 Expected / Anti-patterns 餵給被測 subagent（等於把答案給考生）
- NEVER 未報成本、未經確認就跑 `all`
</fatal_implications>
