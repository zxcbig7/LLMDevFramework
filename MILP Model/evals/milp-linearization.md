---
id: milp-linearization
domain: MILP Model
rule_refs: ["linearization-patterns Fixed-charge", "Big-M 取值鐵律", "pattern tag"]
created: 2026-07-05
updated: 2026-07-05
---

# Eval Case — milp-linearization

## Input（模擬使用者輸入）

模型其他部分我確認過了，只差這條規則，幫我寫成數學式（LaTeX）並標 pattern：工廠可以選擇要不要租某台機器，每台機器每月租金 5000 元；只有租了那台機器，才能用它生產，且每台機器每月產能上限 200 件。

## Expected（可觀察判準，全過才 PASS）

- [ ] 認出這是 **Fixed-charge / Conditional Activation** pattern（有標 pattern tag）
- [ ] 引入 binary 變數（語意名如 `Rent` / `Open`）表示是否租用
- [ ] 有 **linking 約束** `$Produce \le M \cdot Rent$`（開關與產量綁定），M 取產能上限 200（或明確的合法上界）
- [ ] 租金以 `$5000 \cdot Rent$` 進**目標函數**（不是憑空加常數）
- [ ] 數學式為 LHS op RHS 原形、Big-M 值有依據且建議寫成具名 PARAM

## Anti-patterns（出現任一即 FAIL）

- 只把租金 5000 加進目標、卻**漏掉 linking 約束**（開關與產量沒綁 → 可不租照生產 / 租了不生產照付但無意義）
- Big-M 用 `99999` 之類 magic number，或沒說明 M 的依據
- 用 `if...then`、`abs()`、變數相乘 / 相除等非線性寫法
- 把「是否租用」當連續變數或當已知 parameter，而非 binary decision variable
- 預先移項化簡（如寫成 `$Produce - 200\,Rent \le 0$`）
</content>
