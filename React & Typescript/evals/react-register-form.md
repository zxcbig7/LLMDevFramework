---
id: react-register-form
domain: React & Typescript
rule_refs: ["React Hook Form", "Zod"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — react-register-form

## Input（模擬使用者輸入）

做一個註冊表單 component：email、密碼、確認密碼，要驗證格式（email 格式、密碼至少 8 碼、兩次密碼一致），送出打 /api/register

## Expected（可觀察判準，全過才 PASS）

- [ ] 複雜表單用 React Hook Form + Zod resolver（schema 定義驗證規則）
- [ ] 送出走共用 axios mutation helper / client（`src/lib/api`），不在 component 內自建 axios instance 或手刻 fetch
- [ ] 所有 function / component 有 explicit return type，全程無 `any`
- [ ] component 名 PascalCase、props 用 interface（無 `I` 前綴）

## Anti-patterns（出現任一即 FAIL）

- component 內 `axios.create()` 或直接 `fetch('/api/register', ...)`（繞過共用 client）
- 手刻 useState 逐欄位驗證取代 schema（此題屬複雜表單，規範走 RHF + Zod）
- 出現 `any`
