# PowerShell — .ps1 腳本開發規範

<system_context>
Windows PowerShell `.ps1` 開發、review、debug 守則。與 CMD Developer（`.bat`）並列為 sibling。
適用：Windows 11，預設相容 **Windows PowerShell 5.1**（cp950 環境），標注 PowerShell **7+**(pwsh) 專屬語法。
何時用 `.ps1` vs `.bat` 見下方 `<selection_guide>`。共通的 CRLF / encoding 底層原理見
`CMD Developer/CLAUDE.md` 雷區 5、9，本檔只列 PowerShell 專屬差異。
</system_context>

<critical_notes>
- MUST 腳本開頭 `Set-StrictMode -Version Latest` + `$ErrorActionPreference = 'Stop'` —— Why: PS 預設吞 non-terminating error 繼續跑，不設等於沒有錯誤處理
- MUST 含非 ASCII 的 `.ps1` 存 **UTF-8 with BOM** —— Why: 5.1 用 ANSI(cp950) 讀無 BOM 檔，繁中亂碼 / 解析失敗；UTF-8 BOM 是 5.1 + 7 都安全的唯一交集
- MUST 寫檔明確帶 `-Encoding`（`Set-Content` / `Out-File`）—— Why: 5.1 預設 ANSI、7 預設 utf8NoBOM，不指定產出編碼不可預測
- MUST `.gitattributes` 加 `*.ps1 text eol=crlf`（同 `.bat`，見 CMD Developer 雷區 9）
- MUST 腳本標目標版本 `#requires -Version 5.1` —— Why: 7+ 語法在 5.1 直接 syntax error，先擋住
- MUST 參數用 `param()` + 型別 + `[CmdletBinding()]` —— Why: 取代手動解析 `$args`，自帶驗證 / `-Verbose` / `-WhatIf`
- MUST 比較用 `-eq -ne -lt -gt`，且 `$null` 放左邊（`if ($null -eq $x)`）—— Why: PS 無 C 式 `==`；`$null` 放右邊遇陣列會誤判
- NEVER 用 `Write-Host` 輸出資料 —— ALWAYS `Write-Output` / `return` —— Why: `Write-Host` 直吐 console，pipeline / 變數接不到
（注入類絕對禁令見 `<fatal_implications>`）
</critical_notes>

<file_map>
PowerShell/CLAUDE.md - 本檔（.ps1 規範 + .bat vs .ps1 選用決策）
（本 domain 暫不附 slash command；`.ps1` 靠 root Router 自動路由套用）
</file_map>

<paved_path>
**標準骨架**（複製即用）

```powershell
#requires -Version 5.1
[CmdletBinding()]
param(
    [Parameter(Mandatory)][string]$Path,
    [int]$Retry = 3
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$scriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path

function Invoke-Work {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Target)
    # ... 邏輯，回傳物件而非 Write-Host ...
}

try {
    Invoke-Work -Target $Path
    Write-Output 'OK'
}
catch {
    Write-Error "失敗：$($_.Exception.Message)"
    exit 1
}
```

**呼叫外部程式並檢查錯誤**（cmdlet 的 `$ErrorActionPreference` 管不到外部 exe）

```powershell
& git @('clone', $repo, $dest)
if ($LASTEXITCODE -ne 0) { throw "git clone 失敗 ($LASTEXITCODE)" }
```
</paved_path>

<selection_guide>
**`.bat` vs `.ps1` 何時用哪個**

| 情境 | 選 |
|------|----|
| 雙擊執行 / 不能依賴 execution policy / 相容老系統 | `.bat`（見 CMD Developer）|
| 物件、結構化錯誤處理、REST / JSON、複雜邏輯 | `.ps1` |
| 一次性 one-liner | `pwsh -Command "..."`，**不開檔** |
| 跨平台（Linux / Mac）| `.ps1` + PowerShell 7（pwsh）|

原則：**能寫 `.ps1` 優先 `.ps1`**；`.bat` 只在上表硬限制時用。
</selection_guide>

<patterns>
### 雷區 1：編碼（PowerShell 專屬差異）
- ⚠️ **5.1 與 7 的 `-Encoding utf8` 意義不同**：5.1 = with BOM；7 = **no** BOM。要 BOM 明確：7 用 `utf8BOM`
- 寫含中文 → 5.1：`Set-Content -Encoding UTF8`；7：`Set-Content -Encoding utf8BOM`
- console 中文亂碼 → `[Console]::OutputEncoding = [Text.Encoding]::UTF8`（必要時 `chcp 65001`）
- 編輯器 / 工具重存掉 BOM 的補救：

```powershell
$c = Get-Content .\x.ps1 -Raw -Encoding UTF8
Set-Content .\x.ps1 -Value $c -Encoding utf8BOM -NoNewline   # 5.1 用 -Encoding UTF8
```

### 雷區 2：錯誤處理（terminating vs non-terminating）
- cmdlet 錯誤預設 non-terminating，**不進 catch** → 靠全域 `$ErrorActionPreference='Stop'` 或 `-ErrorAction Stop`
- 外部 exe（git / docker / minikube）**不吃** `$ErrorActionPreference` → 自己檢 `$LASTEXITCODE`

❌ Bad：`Remove-Item $p; Write-Output done`（$p 不存在也照吐 done）
✅ Good：`try { Remove-Item $p -ErrorAction Stop } catch { ... }`

### 雷區 3：pipeline / 物件思維（別 text-grep）
❌ Bad：`Get-ChildItem | Out-String | Select-String '\.log'`
✅ Good：`Get-ChildItem -Filter *.log`、`... | Where-Object Length -gt 1MB`

### 雷區 4：`$null` / 空值
- `if ($null -eq $x)`（$null 放左）；空陣列 `@()` 在 `if` 為 false；單元素陣列會被 unwrap

### 雷區 5：path / 引號
- 路徑用 `Join-Path`，別字串拼 `\`
- 字面用單引號 `'literal'`；內插用 `"$var"`；含運算式 `"$($obj.Prop)"`
- 傳參給外部程式用陣列：`& $exe @args`，別自己加引號

### 雷區 6：版本差異 5.1 vs 7+
| 語法 | 5.1 | 7+ |
|------|-----|----|
| `$x?.Prop` / `??` / `??=` | ❌ | ✅ |
| `ForEach-Object -Parallel` | ❌ | ✅ |
| 三元 `a ? b : c` | ❌ | ✅ |
| `ConvertFrom-Json -AsHashtable` | ❌ | ✅ |
| 預設寫檔編碼 | ANSI / UTF8(BOM) | utf8NoBOM |

→ 跨版本腳本避用 7 專屬語法；只跑 7 就標 `#requires -Version 7`。

### 雷區 7：execution policy fallback（不能跑 .ps1 時）
- 公司鎖 Restricted → `.ps1` 直跑被擋
- fallback：① 指令貼進 PS 視窗**互動執行**（不受 policy）② `powershell -ExecutionPolicy Bypass -File x.ps1`（單次，留意公司政策）③ 改讓 AI 直接以 tool 操作完成（本框架 BOOTSTRAP 哲學：能不跑腳本就不跑）
- NEVER 教人全域 `Set-ExecutionPolicy Unrestricted`（降全機防線）
</patterns>

<common_tasks>
- **寫新 .ps1** → 用 `<paved_path>` 骨架起手
- **跑既有 .ps1** → `pwsh -File x.ps1`（7）/ `powershell -File x.ps1`（5.1）
- **含中文存檔亂碼** → 用雷區 1 補救一行重存 BOM
- **從 .bat 呼叫 PS** → 見 `CMD Developer/CLAUDE.md` 雷區 6
- **參考實作** → `Mermaid Diagrams/references/render-mermaid.ps1`
</common_tasks>

<hatch>
- 一次性指令 → `pwsh -Command "..."`，別開 `.ps1`
- 環境不能跑 PS → 改 `.bat`（見 CMD Developer）
- 需 GUI → `Add-Type` 拉 WinForms/WPF；邏輯一複雜就別用 PS
</hatch>

<fatal_implications>
- NEVER `Remove-Item -Recurse -Force` 配未驗證變數（空值會掃整個 drive）
- NEVER 明文存 password / token → 用 SecretManagement / `Get-Credential` / 環境變數
- NEVER `Invoke-Expression` 或 `iwr ... | iex` 吃外部 / 不可信字串（= eval 注入）
- NEVER 全域改 `ExecutionPolicy` 為 Unrestricted（降低整機防線）
</fatal_implications>
