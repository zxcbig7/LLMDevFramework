---
id: milp-naming-hardcode
domain: MILP Model
rule_refs: ["禁止 Hardcode", "AddLHS"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — milp-naming-hardcode

## Input（模擬使用者輸入）

模型我已經確認過了，直接寫 code。這是 Model.md 的 Capacity 約束，幫我寫對應的 Constraint 類別：

$$\sum_{g \in GlassType} UsageRate_g \cdot Produce_g \le MachineCapacity$$

其中 UsageRate：Regular = 0.5、Tempered = 0.75；MachineCapacity = 40。

## Expected（可觀察判準，全過才 PASS）

- [ ] 類別名為 `Constraint_Capacity`（或 Capacity 語意核心 + `Constraint_` 前綴），變數用 `VariableX_Produce` 類命名
- [ ] 係數 0.5 / 0.75 / 40 全部經 Parameter 的 `QTY` 欄位（透過 dataload）取得，先 LINQ 存局部變數再傳入
- [ ] Model 左側項用 `AddLHS`、右側用 `AddRHS`，比較方向用 `CreateLessEqual`
- [ ] 約束命名字串含語意名稱（如 `"Capacity"` 或 `"MachineCapacity"`）

## Anti-patterns（出現任一即 FAIL）

- `AddLHS(0.5, ...)`、`AddRHS(40)` 等裸數字直接進呼叫（未經 Parameter / dataload）
- LINQ 查詢直接內嵌在 `AddLHS(...)` / `AddRHS(...)` 參數裡
- 移項、改號、或用 `CreateGreatEqual` / `CreateEqual` 取代 `<=`
- 發明不存在的 API（如 `GetVarSol`、`addPool`）
