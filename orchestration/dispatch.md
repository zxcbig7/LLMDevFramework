# Dispatch — 模型調度守則

<system_context>
主對話（指揮官）如何派 subagent：何時派、派給誰、用什麼 model/effort、怎麼驗收。
立論依據：`diagnosis-2026-07-08.md §Top-1`（指揮官下場是最大 token 洩漏與失焦主因）。
判斷類問題（何時升級、何時算完成、何時問使用者）見 `judgment.md`；派工 prompt 直接套 `templates/`。
</system_context>

<critical_notes>
- NEVER 主對話自己掃 repo / 讀一堆檔案找答案 —— ALWAYS 派 subagent，主對話只收結論
  Why: 原始材料灌進主 context，規則被稀釋，之後每一步判斷都變差（弱模型尤其嚴重）
- NEVER 派工不寫驗收條件 —— ALWAYS 三件套齊全才 spawn（目標與動機、驗收條件、回報格式）
  Why: 沒有驗收條件的派工，回來的東西無法判對錯，只能憑感覺收貨
- NEVER 產出者自己宣告驗收通過 —— ALWAYS 派 fresh-context `verifier`
  Why: generator 對自己的產出必然樂觀（code-review command 已載明此理）
- NEVER subagent 把長產物貼回主對話 —— ALWAYS 落檔傳路徑 + ≤10 行摘要
</critical_notes>

## 已查證事實（2026-07-08，出處為 Claude Code 官方文件）

| 事實 | 內容 | 出處 |
|---|---|---|
| Agent tool 每次呼叫 | 可指定 `model`（sonnet / opus / haiku / 完整 model id）；**不能**指定 effort | code.claude.com/docs/en/sub-agents.md |
| effort 控制點 | 只有兩處：settings.json `effortLevel`（主對話，值 low/medium/high/xhigh）與自訂 agent frontmatter `effort`（值 low/medium/high/xhigh/max，依模型而定） | configuration.md + sub-agents.md |
| 自訂 agent | `~/.claude/agents/*.md`，frontmatter 支援 `name/description/tools/model/effort` 等；`subagent_type` 用其 name 呼叫 | sub-agents.md |
| model 解析優先序 | env `CLAUDE_CODE_SUBAGENT_MODEL` > 呼叫參數 `model` > agent frontmatter `model` > 主對話 model | sub-agents.md |
| 別名對應（2026-07） | `sonnet`=Sonnet 5、`opus`=Opus 4.8、`haiku`=Haiku 4.5；別名隨版本指向最新 | model-config.md |
| slash command | frontmatter 也支援 `model` 欄位（執行期間覆蓋） | skills.md |

> `fable` 別名 2026-07-08 之後不可用，NEVER 在派工時指定它。

## 模型階梯

| 別名 | 定位 | 用在 |
|---|---|---|
| `haiku` | 便宜量產 | 已解出 pattern 的批次套用（`batch-worker`）、粗掃盤點 |
| `sonnet` | 預設 workhorse | 搜尋、實作、重構、研究、驗收（`verifier`）——**沒有明確理由就用這個** |
| `opus` | 升級位 | 架構選型、模糊題、除錯卡關、第二意見（`second-opinion`） |

## 何時派、派給誰

**必派 subagent（主對話 NEVER 自己做）：**

| 情境 | 派給 | model |
|---|---|---|
| 掃 repo / 盤點結構 / 「哪裡有 X」 | `Explore` | 呼叫參數給 `sonnet`（範圍小給 `haiku`） |
| 要讀 3 個以上檔案且不確定哪個相關 | `Explore` | `sonnet` |
| 網路研究 / 查文件 | `general-purpose`，結果落檔 | `sonnet` |
| 產 CodeMap / 產長文件 | `general-purpose`，落檔回路徑 | `sonnet` |
| 3 檔以上同型修改（pattern 已定案、有 before/after 範例） | `batch-worker` | （定義內建 haiku） |
| 同型修改但 pattern 還沒解出 | 先派 `general-purpose` 解第一個，成功後轉上一列 | `sonnet` |
| 驗收任何交付物 | `verifier` | （定義內建 sonnet/high） |
| 高風險判斷 / 方案評審 | `second-opinion` | （定義內建 opus/xhigh） |
| Claude Code / API 用法問題 | `claude-code-guide` | 預設 |

**主對話自己做（派工反而虧——spawn 有 cold-start 成本）：**
- 讀 1–2 個已知路徑、<300 行的檔
- 單檔精準編輯、已知指令執行（git status、build）
- 與使用者的對話、決策、彙整

## 派工三件套（每次 spawn 的 prompt 必含）

1. **目標與動機**：做什麼 + 為什麼（subagent 看不到主對話，沒有動機就沒有邊界判斷力）
2. **驗收條件**：逐條、可驗證（會被 verifier 拿去逐條打分的那種）
3. **回報格式**：規定結構與長度上限

✅ Good：
```
目標：找出本 repo 所有直接 new HttpClient() 的位置（動機：要統一改成 IHttpClientFactory，先盤點影響面）。
驗收：(1) 列出每處 檔案:行號 (2) 標注各處是否在 DI 可及範圍 (3) 沒有遺漏——附你的搜尋 pattern 供查核。
回報：表格（檔案:行號｜DI 可及？｜備註），≤20 行；不要貼程式碼原文。
```
❌ Bad：`幫我看看專案裡 HttpClient 怎麼用的`（無動機、無驗收、無格式 → 回來一篇散文，還得重派）

## 回報合約（subagent 端）

- 只回：結論、判斷、`檔案:行號`；上限 30 行（個別模板另有規定者，以模板為準）
- 產物超過 50 行（CodeMap、研究筆記、報告）→ 落檔到派工方指定路徑，回傳路徑 + ≤10 行摘要
- 證據引用 ≤5 行原文；NEVER 整檔貼回
- 派工 prompt 裡就要把這段寫進去（模板已含，見 `templates/`）

## 升降級路徑

判定「錯」的標準 = verifier FAIL、或主對話拿驗收條件逐條對照失敗。感覺不算。

1. **haiku 錯 1 次** → 直接升 `sonnet` 重派（不給 haiku 第二次機會，重試同 model 是浪費）
2. **sonnet 同一子任務連錯 2 次** → 升 `opus`，且 MUST 附完整失敗軌跡：
   原派工 prompt、兩次輸出摘要、每次 FAIL 在哪條驗收條件
   Why: 不附軌跡，opus 會重走一遍相同的死路
3. **opus 解出、且同 pattern 還有多份要套** → 把解法寫成 before/after 範例，降級派 `batch-worker`
4. **同一子任務總重試上限 2 輪**（不含首次）→ 到頂還 FAIL：停，走 `judgment.md §換路訊號`
   （通常代表任務定義有問題，不是執行力問題——再重試只是燒 token）

## 驗證不自驗

| 交付物 | 驗法 |
|---|---|
| 檔案落地（文件、設定、規範） | `verifier` read-back：存在、完整、無 placeholder、內部引用有效 |
| 程式碼 | 優先測試 / build / 實跑（`/verify`）；跑不了才 `verifier` 讀 diff 對照驗收條件 |
| 高風險 / 不可逆 / 模糊題 | `second-opinion` 裁決；或生成 2–3 個候選方案交它評審選優 |
| 主對話與 verifier 意見相左 | `second-opinion` 仲裁，仍相左 → 問使用者 |

## Fallback

Agent tool 不可用或被拒 → 自己做，但輸出第一行 MUST 標明：
`⚠️ 未派工（Agent 不可用）——本結果含自驗，重要決策建議另開 session 覆核`
NEVER 默默自己做完當作有派過。
