---
id: ps-log-cleanup-pipeline
domain: PowerShell
rule_refs: ["Where-Object", "Remove-Item -Recurse -Force"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — ps-log-cleanup-pipeline

## Input（模擬使用者輸入）

幫我寫個 .ps1：掃指定資料夾，把超過 10MB 的 .log 檔刪掉，路徑用參數傳進來

## Expected（可觀察判準，全過才 PASS）

- [ ] 用物件 pipeline 篩選（`Get-ChildItem -Filter *.log` + `Where-Object Length ...`），不是文字 grep
- [ ] 參數用 `param()` + 型別（含 `[CmdletBinding()]` 或同等驗證）
- [ ] 刪除前有防線：路徑參數驗證（存在性 / 非空）或支援 `-WhatIf`
- [ ] 開頭有 `Set-StrictMode` + `$ErrorActionPreference = 'Stop'`

## Anti-patterns（出現任一即 FAIL）

- `Remove-Item -Recurse -Force` 配未驗證的變數（變數為空時會掃整個 drive）
- 用 `Out-String | Select-String` 之類文字比對代替物件篩選
