---
id: milp-solution-verify
domain: MILP Model
rule_refs: ["解驗證協定", "Status 三分診斷"]
created: 2026-07-05
updated: 2026-07-05
---

# Eval Case — milp-solution-verify

## Input（模擬使用者輸入）

我的模型 `dotnet build` 過了，`Solve()` 回傳 `Status == Optimal`，目標值 12500。這樣就算完成了嗎？

## Expected（可觀察判準，全過才 PASS）

- [ ] 不因為 `Optimal` 就宣稱完成——明確指出 Optimal 只保證 solver 解了「它拿到的模型」，不保證那模型 = 題目
- [ ] 提出解驗證協定：至少涵蓋「解值**代回每一條** constraint 確認可行」與「目標值/關鍵變數的**單位與量級**對照題目」
- [ ] 提到 LP relaxation bound sanity（整數解落在 relaxation bound 對的一側）或對照 Model.md 手算小例
- [ ] 用可觀察動作描述（跑哪些檢查），非「感覺應該對」

## Anti-patterns（出現任一即 FAIL）

- 直接回「Optimal 就是完成了 / 沒問題」
- 只看目標值 12500，不提代回約束或單位檢查
- 完全不提任何驗證步驟就結案
- 把驗證跳過、直接建議進 Phase 3 效能 tuning
</content>
