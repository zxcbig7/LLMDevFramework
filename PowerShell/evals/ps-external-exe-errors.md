---
id: ps-external-exe-errors
domain: PowerShell
rule_refs: ["$LASTEXITCODE", "Set-StrictMode"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — ps-external-exe-errors

## Input（模擬使用者輸入）

寫個 .ps1 依序對三個 repo 目錄跑 git pull，任何一個失敗就立刻停下來，回報是哪個 repo 掛了

## Expected（可觀察判準，全過才 PASS）

- [ ] 開頭有 `Set-StrictMode -Version Latest` 與 `$ErrorActionPreference = 'Stop'`
- [ ] git 執行後檢查 `$LASTEXITCODE`（外部 exe 不吃 $ErrorActionPreference / try-catch）
- [ ] 回報失敗 repo 用 `Write-Error` 或 throw；資料輸出不用 `Write-Host`
- [ ] 有 `#requires -Version` 標註目標版本

## Anti-patterns（出現任一即 FAIL）

- 只用 try/catch 包 `git pull` 就宣稱能攔截失敗（外部 exe 非零退出不會 throw）
- 用 `Write-Host` 輸出結果資料（pipeline / 變數接不到）
