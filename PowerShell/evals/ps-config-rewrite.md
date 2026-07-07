---
id: ps-config-rewrite
domain: PowerShell
rule_refs: ["UTF-8 with BOM", "#requires -Version 5.1"]
created: 2026-07-03
updated: 2026-07-04
---

# Eval Case — ps-config-rewrite

## Input（模擬使用者輸入）

幫我寫個 .ps1 讀 appsettings.json、把連線字串換成新值後存回，註解用中文寫

## Expected（可觀察判準，全過才 PASS）

- [ ] 腳本開頭有 `#requires -Version 5.1`（或明確標註目標 PowerShell 版本）
- [ ] 讀檔與寫檔都明確指定編碼（cmdlet 帶 `-Encoding`，或 .NET API 傳明確 encoding 物件如 `UTF8Encoding($false)`）
- [ ] 有提醒：本 .ps1 含中文註解，檔案本身需存 UTF-8 with BOM（5.1 相容）

## Anti-patterns（出現任一即 FAIL）

- `Set-Content` / `Out-File` 寫檔未帶 `-Encoding`（5.1 預設 ANSI、7 預設 utf8NoBOM，產出不可預測）
- 標示 5.1 相容卻使用 7+ 專屬語法（`?.`、三元運算子、`-AsHashtable` 等）
