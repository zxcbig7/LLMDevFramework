# Harness 診斷 — 2026-07-08（Fable 5 一次性 session）

<system_context>
本檔是 orchestration domain 的立論基礎：盤點當日 harness 實況後，找出最漏 token、最易失焦、最易出錯的前三名，各附修法。
後續文件（dispatch.md / judgment.md / maintenance.md / 全域 CLAUDE.md 改版）皆引用本檔，不重複論證。
證據等級標注：【實測】= 本 session 直接觀察；【文件】= 官方文件查證（出處在 dispatch.md 事實表）；【推估】= 合理推論，未量測。
</system_context>

## Top 1：指揮官下場 —— 最大的 token 洩漏，也是失焦主因

**現象**：主對話自己做大量讀取（掃 repo、產 CodeMap、讀大檔、查網頁、批次改檔），
原始材料直接灌進主 context。

**證據**：
- 【實測】全域 CLAUDE.md 的 CodeMap 規則要求「進陌生模組 / review 前先產 CodeMap.md」，
  而產 CodeMap = 逐檔 Grep + Read。主對話自己跑，一個中型 repo 就是數萬 token 進 context（推估）。
- 【實測】`/code-review` 已把 review 本體派給獨立 evaluator subagent（正確示範），
  但 Phase 1 的 CodeMap 掃描仍在主對話執行。
- 【實測】`~/.claude/agents/` 不存在——沒有任何自訂 subagent，
  等於沒有可預設 model + effort 的派工單位。

**為何傷害雙倍**：token 花掉只是帳面成本；更貴的是主 context 被原始材料稀釋後，
CLAUDE.md 的規則離注意力越來越遠，後續每一步的判斷品質都下降。弱模型受此影響遠大於強模型。

**修法**（已落地為制度）：
1. `orchestration/dispatch.md` 鐵則：掃 repo / 讀 3 個以上檔案自己不確定哪個相關 /
   查網頁 / 批次改檔 → 一律派 subagent，主對話只收結論與 `檔案:行號`。
2. 部署自訂 agents（`verifier` / `batch-worker` / `second-opinion`）補上 effort 控制缺口。
3. CodeMap 這類「產大檔」任務：subagent 落檔，回傳路徑 + 10 行內摘要，全文不進主對話。

## Top 2：設定殘留 —— 每個 session 開場固定漏

**逐項盤點（`~/.claude/settings.json` + `~/.claude/mcp.json`，2026-07-08 實測）**：

| 項目 | 問題 | 修法 |
|---|---|---|
| `"model": "claude-fable-5[1m]"` | 本 session 後 fable 不可用，寫死會失效 | 改 `opusplan` 或 `sonnet`（使用者決定，見 LETTER.md） |
| mcp.json `notion` server | `Authorization: Bearer AAA` 是 placeholder，每 session 啟動一個永遠 auth 失敗的 npx process；且與 claude.ai Notion connector 功能重複 | 刪 local `notion` entry（動手前先問使用者） |
| mcp.json `playwright` | 全域常駐。本 harness 有 ToolSearch 延遲載入、token 成本低；CLI harness 會全量載入 20+ tool schema | 只在需要瀏覽器的專案放 project-level `.mcp.json` |
| `C:\Users\zxcbi\CLAUDE.md`（4 行） | 內容（PowerShell 指令風格）與全域 CLAUDE.md 重複；因 `additionalDirectories` 含整個 home dir，每 session 多載一次 | 內容已併入全域，此檔可刪（先問使用者） |
| `additionalDirectories: ["C:\\Users\\zxcbi", ...]` | 整個 home dir 入 scope，過寬 | 收斂成實際需要的子目錄 |
| deny list `Bash(rm *)` 等 | prefix match 擋不住 `cd x && rm -rf .`、`powershell Remove-Item` | 別當安全網用；真防線是「刪檔前列清單問使用者」的行為規則（judgment.md §3） |

## Top 3：規則以「記憶」存在，不是以「檢查點」存在 —— 弱模型最易出錯處

**現象**：全域 + 專案 CLAUDE.md 累計 200+ 行 MUST/NEVER，依賴模型「一直記得」。
context 越長、規則離當前 prompt 越遠，遵從率越低（無精確量測，但有實例）。

**證據**：
- 【實測】2026-07-03 BOM 事件：「NEVER 用 Out-File 寫檔」規則在 context 裡，仍炸掉 4 個 command 檔
  ——規則在場 ≠ 規則被執行。
- 【實測】全域 CLAUDE.md 的 dispatch 表 + CodeMap 規則 + Code Defaults 全是「散裝 MUST」，
  沒有執行時的強制卡點。

**修法**（優先序）：
1. **規則就近化**：把「何時做什麼」做成入口 command / skill（`/code-review` 模式：
   規則寫在 command 裡，跑的時候才載入，不佔常駐 context 也不會忘）。
2. **驗證外部化**：「完成」的宣告不靠自覺，靠 fresh-context `verifier` agent 對照
   judgment.md 的 Definition of Done checklist。自驗必然樂觀（code-review command 已載明此理）。
3. **行為回歸測試**：規範改動後跑 `/harness-eval <domain>`。orchestration domain 依
   maintenance.md 規定補 evals。

## 引用錨點

後續文件引用格式：`diagnosis-2026-07-08.md §Top-1`（指揮官下場）、`§Top-2`（設定殘留）、`§Top-3`（記憶 vs 檢查點）。
