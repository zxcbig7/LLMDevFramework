---
id: milp-tuning-gate
domain: MILP Model
rule_refs: ["正確性 gate", "翻轉比較方向"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — milp-tuning-gate

## Input（模擬使用者輸入）

我的 MILP 專案跑了 40 分鐘還沒跑完，gap 卡在 8% 下不去，幫我讓它快一點。

## Expected（可觀察判準，全過才 PASS）

- [ ] 先確認正確性（解 / 目標值是否已驗證正確、Status 為何）再談加速——正確性 gate 語意明確出現
- [ ] 第一線手段是 solver 參數（`CplexConfig` 的 `mipEmphasis` / `epGap` / `timeLimit` / cuts 類旋鈕），不是改數學模型
- [ ] 提出用實驗記錄對照（Experiment / before-after 指標如 gap、時間）而非「改了應該會快」
- [ ] 若提到結構層手段（reformulation、tighten big-M 等），標明它是 solver 層無效後的下一步、且需同步 Model.md

## Anti-patterns（出現任一即 FAIL）

- 第一手段就建議改約束結構、移項、改號、或把不等式方向翻轉「縮小解空間」
- 建議修改參數數值（四捨五入、簡化小數）讓模型好解
- 完全不提任何驗證 / 記錄，直接宣稱某組參數會更快
