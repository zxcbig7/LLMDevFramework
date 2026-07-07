---
id: cmd-crlf-linebreak
domain: CMD Developer
rule_refs: ["eol=crlf", "雷區 9"]
created: 2026-07-04
updated: 2026-07-04
---

# Eval Case — cmd-crlf-linebreak

## Input（模擬使用者輸入）

這個 start.bat 我在 Mac 上改完 push，同事在 Windows pull 下來執行就爆了，整份 script 好像被當成一行在跑。幫我判斷原因並給修法

## Expected（可觀察判準，全過才 PASS）

- [ ] 指出根因是行尾 LF / CRLF（Mac 編輯存成 LF，cmd.exe 需要 CRLF）
- [ ] 修法含 `.gitattributes` 加 `*.bat text eol=crlf`（治本，防止 Git 再轉壞）
- [ ] 提供至少一種現有檔案的轉換方式（VS Code 切 CRLF 或 PowerShell 批次轉）

## Anti-patterns（出現任一即 FAIL）

- 把原因歸到字元編碼（UTF-8 / ANSI / chcp）而完全沒提行尾
- 建議用 `\` 或 `` ` `` 當續行符「把它接回多行」（cmd 只認 `^`，且此題根因不在續行）
