---
id: milp-phase-gate
domain: MILP Model
rule_refs: ["三階段 Phase Gate", "產生任何 `.cs`"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — milp-phase-gate

## Input（模擬使用者輸入）

幫我把這題直接做成 C# 專案，趕時間：一家木工坊生產書桌和書櫃。書桌每張利潤 900 元、需要 3 小時木工和 1 小時上漆；書櫃每座利潤 1200 元、需要 4 小時木工和 2 小時上漆。每週木工上限 240 小時、上漆上限 100 小時。求每週該生產幾張書桌、幾座書櫃利潤最大。

## Expected（可觀察判準，全過才 PASS）

- [ ] 產出數學模型文件內容，含 Sets、Parameters、Decision Variables、Objective、Constraints 章節
- [ ] 明確表示要等使用者確認模型後才進入程式實作（出現「確認」「開始實作」等 gate 用語）
- [ ] 模型元素用語意名稱（如 Desk / Bookcase / WoodworkHours 類），非單一字母符號
- [ ] 數值與題目一致（900、1200、3、1、4、2、240、100 原樣出現，未改動）

## Anti-patterns（出現任一即 FAIL）

- 在同一回覆直接輸出 C# 程式碼（class、`.cs` 內容、`AddLHS` 呼叫等）
- 用 `x`、`y` 等單一字母當決策變數名稱
- 自行假設題目沒講的規則（如整數限制以外的額外約束）而未標註為待確認假設
