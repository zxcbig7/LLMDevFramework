---
id: milp-naming-hardcode
domain: MILP Model
rule_refs: ["禁止 Hardcode", "AddLHS", "結構（天條）", "Build(OptEngine engine)"]
created: 2026-07-04
updated: 2026-08-02
---

# Eval Case — milp-naming-hardcode

## Input（模擬使用者輸入）

模型我已經確認過了，直接寫 code。這是 Model.md 的 Capacity 約束，幫我寫對應的 Constraint 類別：

$$\sum_{g \in GlassType} UsageRate_g \cdot Produce_g \le MachineCapacity$$

其中 UsageRate：Regular = 0.5、Tempered = 0.75；MachineCapacity = 40。

## Expected（可觀察判準，全過才 PASS）

- [ ] 類別名為 `Constraint_Capacity`（或 Capacity 語意核心 + `Constraint_` 前綴），變數用 `VariableX_Produce` 類命名
- [ ] 係數 0.5 / 0.75 / 40 全部經 Parameter 的 `QTY` 欄位取得，先 LINQ 存局部變數再傳入
- [ ] Model 左側項用 `AddLHS`、右側用 `AddRHS(常數)`，比較方向用單參數 `CreateLessEqual(name)`
- [ ] 約束命名字串用 `ConstraintName`（或含語意名稱如 `"Capacity"`）
- [ ] 類別宣告為 `sealed`，且有 `<summary>` 貼上對應的數學式
- [ ] 建構子只收 `Set_*` / `List<Parameter_*>` / scalar；`OptEngine` 從 `Build(OptEngine engine)` 傳入

## Anti-patterns（出現任一即 FAIL）

- `AddLHS(0.5, ...)`、`AddRHS(40)` 等裸數字直接進呼叫（未經 Parameter）
- LINQ 查詢直接內嵌在 `AddLHS(...)` / `AddRHS(...)` 參數裡
- 移項、改號、或用 `CreateGreatEqual` / `CreateEqual` 取代 `<=`
- 用 `CreateLessEqual(40.0, name)` 這個 overload 表達右側
- 建構子收 `Dataload` 或 `OptEngine`；`Build()` 不吃 engine 參數
- 建構子參數型別退化成 `IReadOnlyList<string>` 而非 `Set_GlassType`
- 發明不存在的 API（如 `GetVarSol`、`addPool`）
