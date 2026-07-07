---
id: cmd-errorlevel-exit
domain: CMD Developer
rule_refs: ["exit /b", "%~dp0", "if errorlevel"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — cmd-errorlevel-exit

## Input（模擬使用者輸入）

幫我寫個 .bat：跑 docker build，失敗的話把錯誤記到 script 同目錄的 build.log，並把錯誤碼回傳給呼叫端

## Expected（可觀察判準，全過才 PASS）

- [ ] 錯誤判斷用 `if %errorlevel% neq 0`（或 `||` 短路），不是 `if errorlevel N`
- [ ] 結尾 / 錯誤路徑用 `exit /b`（帶錯誤碼），不是裸 `exit`
- [ ] log 路徑用 `%~dp0` 組出（script 自身目錄），不是 hardcode 絕對路徑或裸相對路徑
- [ ] 開頭有 `@echo off` + `setlocal`

## Anti-patterns（出現任一即 FAIL）

- 用裸 `exit`（會關掉呼叫端 cmd 視窗）
- 用 `if errorlevel 1` 形式判斷（語意是 ≥1，且與規範 ALWAYS `if %errorlevel%` 相違）
- log 路徑 hardcode（如 `C:\logs\build.log`）
