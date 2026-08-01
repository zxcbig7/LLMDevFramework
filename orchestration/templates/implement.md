# 派工模板 — 實作

> 用途：新增功能、修 bug、寫 script——會產生或修改程式碼的任務。
> 派給：`general-purpose`；model 呼叫參數給 `sonnet`（符合 `judgment.md §1` 升級條件才 `opus`）。
> 完成後 MUST 另派 `verifier` 驗收（`dispatch.md §驗證不自驗`），不接受實作者自報完成。

## Prompt 模板（複製後填空）

```text
目標：{{做什麼，一句話}}。
動機：{{為什麼做：解什麼問題 / 給誰用}}——邊界情況照這個動機取捨。

背景（你看不到主對話，這是你需要知道的全部）：
- 相關檔案：{{檔案清單 + 各一句話說明}}
- 既有慣例：{{要遵守的 pattern，例：錯誤處理走 ExceptionMiddleware、命名照 domain CLAUDE.md}}
- 專案規範：先讀 {{專案 CLAUDE.md 路徑 / $FW 對應 domain CLAUDE.md}}

範圍鐵則：
- 只動 {{允許修改的檔案 / 目錄}}
- NEVER 動 {{禁區，例：框架本體 / 既有 public API / 其他模組}}
- 看到範圍外的問題 → 記進回報的「額外發現」，不動手

驗收條件（verifier 會逐條打分）：
1. {{可驗證條件 1，例：POST /api/users 對合法 payload 回 201 + Location header}}
2. {{可驗證條件 2，例：對缺 email 的 payload 回 400 且 body 含欄位錯誤}}
3. build 過（{{build 指令}}）+ 既有測試全過（{{test 指令}}）
4. {{新增測試要求，例：新邏輯至少 2 個測試——happy path + 一個邊界}}

回報格式：
- 改了哪些檔（檔案:行號範圍）+ 每檔一句話
- 驗收條件逐條自報 PASS/FAIL + 證據（指令輸出摘錄 ≤5 行）
- 你做的取捨與理由（≤3 條）；額外發現（無則寫「無」）
- 總長 ≤30 行；NEVER 貼整檔內容
```

## 填空要點

- 驗收條件寫成「輸入 → 可觀察的輸出」；寫不出來代表你自己還沒想清楚，先回 `/kg` pre-flight
- 「範圍鐵則」必填——subagent 看不到主對話，不寫禁區它就敢全 repo 亂改
- ❌ 常見錯誤：驗收條件寫「功能正常運作」——verifier 無法打分，等於沒派驗收
