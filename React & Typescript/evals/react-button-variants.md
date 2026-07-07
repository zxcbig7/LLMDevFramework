---
id: react-button-variants
domain: React & Typescript
rule_refs: ["cn()", "cva"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — react-button-variants

## Input（模擬使用者輸入）

幫我做一個 Button 元件，有 primary / secondary / danger 三種樣式，要支援 disabled 狀態

## Expected（可觀察判準，全過才 PASS）

- [ ] 用 Tailwind utility 排版，多 variant 用 `cva` 管理（或至少 `cn()` 合併 className）
- [ ] className 合併走 `cn()`，無手動字串拼接（`a + (x ? ' b' : '')`）
- [ ] 按鈕這類自訂視覺元件用 Tailwind 自建，不是包 antd Button
- [ ] props 用 interface + explicit return type，variant 型別用 union（非裸 string）

## Anti-patterns（出現任一即 FAIL）

- 手動字串拼接 / template literal 拼 className 處理條件樣式
- 引入 antd Button（或第二個元件庫）做純視覺按鈕
- variant prop 型別寫 `string`（該用 `'primary' | 'secondary' | 'danger'`）
