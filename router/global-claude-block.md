<!-- LLMDEVFRAMEWORK:ROUTER START -->
## LLMDevFramework 自動 Dispatch（自適應）

框架根：`{{FRAMEWORK_PATH}}`（下稱 `$FW`）。在**任何專案**工作時，依下表**主動**載入對應規範 / 套用對應 skill，不必使用者手動指定。

**行為規則**

- 主動 dispatch：偵測到對應情境就自動 Read 對應 domain 的 `CLAUDE.md` 或套用 skill，並在回覆用一句話交代「用了什麼、為什麼」。
- 只有「真正分岔且影響大」（架構選型、會動既有 code、有外部副作用、approve production 變更）才問使用者；其餘自行決定。
- domain 規範**讀來參照**，不複製進對話；衝突時專案自身 `CLAUDE.md` 優先於框架。

**語言 / 檔型 → 該 Read 的 domain**

| 偵測 | Read |
|---|---|
| `.sql` / PL/SQL | `$FW/OracleSQL/CLAUDE.md`（10K+ 行 → `/proc-analyze`）|
| `.bat` `.cmd` | `$FW/CMD Developer/CLAUDE.md` |
| `.ps1` | `$FW/PowerShell/CLAUDE.md` |
| `.tsx` `.ts` + `package.json` | `$FW/React & Typescript/CLAUDE.md` |
| `.cs` + `.csproj` | `$FW/.Net Web API/CLAUDE.md` |
| `.cs` + 參考 OptimFoundation.Core/Cplex | `$FW/MILP Model/CLAUDE.md`（優先於 .Net Web API）|
| `.yaml`（k8s/helm/argo/gha）| `$FW/YAML Review/CLAUDE.md` |

**任務情境 → skill / command**

| 情境 | 用 |
|---|---|
| 非 trivial 新功能 | `/sdd` |
| 任務模糊 / 編碼前 | `/kg` pre-flight |
| 「看 code / review」 | `code-review`（先產 `CodeMap.md`）|
| 進新專案 / 無 `CLAUDE.md` | 主動建議 `/scaffold` |
| 改 K8s/Helm/ArgoCD/GHA yaml | `/k8s-review` |
| 要畫圖 | `mermaid-diagrams` skill |
| 做投影片 / 網頁 deck | `guizang-ppt-skill` |
| 改善一段 prompt | `/prompt-improve` |
| 建模 / 最佳化題目（LP/IP/MILP）/ 模型 tuning | `milp-dev` skill（規範：`$FW/MILP Model/CLAUDE.md`）|

**新專案自舉**：進到沒有 `CLAUDE.md` 的專案 → 主動建議 `/scaffold`（偵測技術棧 → 列要寫的檔 → 確認 → 寫 root + 技術獨立子資料夾的 CLAUDE.md，reference `$FW`）。
<!-- LLMDEVFRAMEWORK:ROUTER END -->
