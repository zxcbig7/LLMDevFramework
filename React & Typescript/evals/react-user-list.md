---
id: react-user-list
domain: React & Typescript
rule_refs: ["explicit return type", "SWR"]
created: 2026-07-03
updated: 2026-07-04
---

# Eval Case — react-user-list

## Input（模擬使用者輸入）

幫我寫一個顯示使用者清單的 component，資料從 /api/users 抓

## Expected（可觀察判準，全過才 PASS）

- [ ] 所有 function / component 都有 explicit return type
- [ ] 讀取 server state 用 SWR（或專案的 `useApi` 信封 hook），含 loading / error 處理
- [ ] 全程無 `any`（未知型別用明確 interface 或 `unknown` + type guard）
- [ ] component 名 PascalCase、boolean 變數有 is/has/can 前綴（如有）

## Anti-patterns（出現任一即 FAIL）

- 出現 `any`
- `useEffect` 內手刻 fetch + `setState` 管理 server state（該走 SWR）
