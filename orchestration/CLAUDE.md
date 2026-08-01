# Orchestration — 模型調度與判斷力 domain

<system_context>
主對話（指揮官）如何派 subagent、怎麼驗收、何時升級模型、踩坑教訓怎麼沉澱。
2026-07-08 由 Fable 5 一次性 session 建立，設計給 Sonnet 等級的主模型長期執行。
入口：全域 CLAUDE.md 的 Router「模型調度」段會把多步任務導到本 domain。
</system_context>

<critical_notes>
- NEVER 主對話自己掃 repo / 大量讀檔 / 批次改檔 —— ALWAYS 派 subagent，只收結論（`dispatch.md`）
- NEVER 派工缺三件套（目標與動機、驗收條件、回報格式）—— ALWAYS 套 `templates/` 模板
- NEVER 產出者自己宣告驗收通過 —— ALWAYS 派 fresh-context `verifier`
- MUST 踩坑後當 session 寫進 `lessons.md`（格式在該檔開頭）—— Why: session 結束記憶就沒了
- 改本 domain 檔案前 MUST 先讀 `maintenance.md` 的分區表（綠/黃/紅/凍結）
</critical_notes>

<file_map>
CLAUDE.md                 - 本檔（domain 入口）
diagnosis-2026-07-08.md   - 立論基礎：harness 三大病灶與修法（凍結，歷史文件）
dispatch.md               - 調度守則：何時派誰、model/effort 事實表、三件套、升降級、驗證不自驗
judgment.md               - 判斷 rubric：升級時機、Definition of Done、何時問使用者、換路訊號、品質底線、誠實條款
templates/                - 派工 prompt 模板：search / implement / refactor / research / review（各自足，複製填空即用）
agents/                   - 自訂 agent 定義 source：verifier(sonnet/high)、second-opinion(opus/xhigh)、batch-worker(haiku/medium)——部署到 ~/.claude/agents/
maintenance.md            - 維護協議：分區權限、黃區流程、教訓提升規則、精簡規則
lessons.md                - 踩坑教訓登記簿（append-only）
LETTER.md                 - 給未來 session 的信：待決清單、退化預警（凍結）
</file_map>

<paved_path>
- 多步任務開工 → 讀 `dispatch.md`「何時派、派給誰」表 → 套 `templates/` 派工
- 交付前 → 派 `verifier`（把派工時的驗收條件原封給它）
- 卡關 / 高風險 → `judgment.md` 對應節 → 需要仲裁派 `second-opinion`
- 踩坑 → `lessons.md` 追加一條
- 要改本 domain 任何檔 → `maintenance.md` 分區表先查權限
</paved_path>

<fatal_implications>
- NEVER 在派工 prompt 指定 `fable`（2026-07-08 後不可用）
- NEVER 改 `diagnosis-2026-07-08.md` / `LETTER.md`（凍結；例外見 LETTER §1 的已處理標記）
- NEVER 直接改 `~/.claude/agents/` 部署版——改 `agents/` source 再 copy 過去
</fatal_implications>
