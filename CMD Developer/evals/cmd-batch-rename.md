---
id: cmd-batch-rename
domain: CMD Developer
rule_refs: ["chcp 65001", "EnableDelayedExpansion"]
created: 2026-07-03
updated: 2026-07-04
---

# Eval Case — cmd-batch-rename

## Input（模擬使用者輸入）

幫我寫個 .bat 把資料夾內所有 *.txt 檔名加上今天日期前綴

## Expected（可觀察判準，全過才 PASS）

- [ ] 有處理編碼：輸出含非 ASCII（如中文註解 / 訊息）時 MUST 含 `chcp 65001` 且說明與檔案編碼一致；全 ASCII 輸出則需明確說明編碼考量（如「全 ASCII 無編碼疑慮」）
- [ ] 有 `setlocal` 且啟用 `EnableDelayedExpansion`
- [ ] `for` 迴圈內執行期才賦值的變數用 `!var!` 展開
- [ ] 檔名路徑有引號包裹（處理含空格檔名，如 `"%%~nxf"` / `"!new!"`）

## Anti-patterns（出現任一即 FAIL）

- 迴圈內用 `%var%` 展開執行期才賦值的變數（解析期展開，迴圈裡永遠拿初值）
- `ren` 目標未加引號且未處理含空格檔名
