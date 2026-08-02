# Codex 派工單全集

> 每個 Packet 一段，**整段複製貼進 Codex** 即可。
> 貼之前先確認：Codex 開在「執行位置」欄指定的目錄。
> 四份 spec 已複製到兩個 repo 的 `specs/` 底下，Codex 讀得到。

## 執行順序

```
1a ──→ 1b ──→ 2a ──→ 3 ──→ 4（8 個可平行）
                ├─ 2b ┐
                ├─ 2c ├─ 可與 3、4 平行
                └─ 2d ┘
```

**兩條不可違反**：`2a → 4`（AI 引導沒改，agent 會生舊架構 code）、`3 → 4` 背靠背（中間 AI-Modeling 編不過）。

## 回退點（出事時用）

```powershell
git -C "...\OptimFoundation\OptimFoundation" reset --hard a569d47   # 改前定版 & 註解
git -C "...\AI-Modeling"                     reset --hard 90abae3   # 出事情滾回
```

---

# Packet 1a — 框架：config 分層

**執行位置**：`C:\Users\zxcbi\Desktop\Projects\OptimizationFramework\OptimFoundation\OptimFoundation`

```
你是 C# 框架開發者，任務是執行一份已核准的規格。

【唯一權威來源】
specs/2026-08-01-optimfoundation-dual-config.md
先完整讀完再動手。規格沒寫的事 → 停下回報，NEVER 自行推測。

【工作範圍】只動這些檔：
  src/OptimFoundation.Core/Config/ProjectConfig.cs      ← 新增
  src/OptimFoundation.Core/ISolverEngine.cs             ← ISolverConfig 移除 LogToConsole、LogFilePath
  src/OptimFoundation.Cplex/CplexConfig.cs              ← 移除 6 個非 solver 成員
  src/OptimFoundation.Cplex/OptEngine.cs                ← 新增 ctor overload + 4 行改讀 ProjectConfig
  src/OptimFoundation.Cplex/OptModel.cs                 ← UseConfig overload + ctor 副作用延後到 Execute()
  src/OptimFoundation.Gurobi/GurobiConfig.cs            ← 只做機械修正讓它編得過
  Templates/ 9 個呼叫點 + tests/ 3 個（完整清單見規格 Module Interactions）

【執行方式】依規格 Implementation Plan：先完成 S1–S4（Stub）並確認 build 通過，再進 I1–I8。

【三個已知陷阱】
1. CplexConfig.enableLog 的實際預設是 true（CplexConfig.cs 第 18 行）。
   AI-Modeling/CPLEX_API_REFERENCE.md §4 寫 false 是錯的文件。
   ProjectConfig.EnableSolverLog 預設必須是 true。
2. OptEngine 只有 4 行要改：143、230、238、246。
   下游的 223、253、767 行讀的是私有欄位 _enableLog/_exportLp/_exportMps/_exportSol，不要動。
3. Gurobi 不在開發 scope。ISolverConfig 少兩個成員後，
   把 LogToConsole / LogFilePath 降級為 GurobiConfig 自有屬性即可，不要新增任何功能。

【驗收】
  dotnet build OptimFoundation.sln   → 0 error
  dotnet test                        → 全部通過（現況 69 個）
另外新增兩個測試：① 設定解析順序（ctor 參數優先於 ProjectConfig）
                  ② ConfigSnapshot.From() 產出的 SolverSpecific 不含那 6 個 key

【絕對禁止】
- NEVER 更新 ..\..\AI-Modeling\dlls\ 底下任何 DLL（那是 Packet 3 的工作）
- NEVER 動 AI-Modeling\ 底下任何檔案
- NEVER 改任何模型的數學內容
- build 修 5 次仍失敗 → 停下回報，不要硬修
- 需要動上述清單外的檔案 → 停下回報

【回報】改了哪些檔、build/test 結果、新增測試名稱、任何與規格不符之處。
```

---

# Packet 1b — 框架：runner 對稱化

**執行位置**：同 1a　**前置**：1a 已完成且 build 綠

```
你是 C# 框架開發者，任務是執行一份已核准的規格。

【唯一權威來源】
specs/2026-08-01-optimfoundation-runner-symmetry.md
先完整讀完再動手。規格沒寫的事 → 停下回報。

【這次要做的核心改動】
OptModel 的語意翻轉：
  - 今天的 OptModel（其實是 runner）→ 改名 OptProject
  - OptModel 這個名字讓給「模型定義」物件：AddVariables / AddObjective / AddConstraints
  - 新增 OptExperiment（n 組 solver 設定 × m 個模型的實驗 runner）

【工作範圍】
  src/OptimFoundation.Cplex/OptModel.cs        ← 改寫成模型定義
  src/OptimFoundation.Cplex/OptProject.cs      ← 新增（原 OptModel 的 runner 邏輯）
  src/OptimFoundation.Cplex/OptExperiment.cs   ← 新增
  src/OptimFoundation.Cplex/CplexConfig.cs     ← 加 Clone()
  src/OptimFoundation.Core/Config/ProjectConfig.cs ← 加 Clone()
  src/OptimFoundation.Core/DataContext.cs      ← 加凍結機制
  Templates/ 全部呼叫點（含刪除 Templates/Tutorial/ExperimentRunner.cs、
                        改寫 Templates/Template_CPLEX/CrossExperiment.cs）
  tests/ 全部呼叫點

【執行方式】依規格 Implementation Plan：先完成 S1–S5（Stub）確認 build 通過，再進 I1–I11。

【四個關鍵設計點】
1. AddVariables / AddObjective / AddConstraints 三個階段的執行順序由框架固定
   （variables → objective → constraints），與註冊順序無關。
   這是為了讓「目標式必須先於限制式」不再靠人的紀律。
2. OnSolved 掛在 OptProject，OptExperiment 沒有這個方法（編譯期就找不到）。
3. Clone() MUST 用 (T)MemberwiseClone()，NEVER 手寫逐欄複製。
   CplexConfig 有 40+ 欄位，手寫版在未來新增旋鈕時會靜默漏掉。
   必須配一個反射測試比對 clone 與原件每個 public 成員相等。
4. OptExperiment 的預設與 OptProject 相反：solver log OFF、不匯出、不做 housekeeping。

【驗收】
  dotnet build OptimFoundation.sln   → 0 error
  dotnet test                        → 全綠
  Templates/Sudoku_SHC279、Templates/FJSP_BASIC_BRICK、Templates/Tutorial
    → dotnet run 的 Status 與目標值必須與本次改動前一致（改動前先各跑一次記下來）
  Templates/Template_CPLEX/CrossExperiment 改用 OptExperiment 後
    → 6 個 Trial 的 label 與目標值與手寫版一致

【絕對禁止】
- NEVER 更新 ..\..\AI-Modeling\dlls\、NEVER 動 AI-Modeling\
- NEVER 改任何模型的數學內容
- build 修 5 次仍失敗 → 停下回報

【回報】改了哪些檔、新增/刪除的檔、build/test 結果、三個 Templates 的前後目標值對照。
```

---

# Packet 2a — 文件 Tier 1（AI 引導，15 份）★ 是 Packet 4 的前置

**執行位置**：分兩次跑
- 第一次：`...\OptimizationFramework\AI-Modeling`（11 份）
- 第二次：`...\Projects\LLMDevFramework`（4 份）

```
你是技術文件維護者，任務是把過時的 AI 引導文件更新到新 API。

【唯一權威來源】
specs/2026-08-01-optim-docs-and-projects-migration.md
  §檔案清單 → Tier 1
  §before / after pattern
  §前置：一件已完成但文件未同步的事
搭配 specs/2026-08-01-optimfoundation-dual-config.md 與
     specs/2026-08-01-optimfoundation-runner-symmetry.md 查新 API 正確寫法。

【為什麼這包最重要】
這 15 份是 AI 引導文件（Prompts / claudemdTemplate / phase 規範）。
後續會有 agent 讀它們去遷移 8 個練習專案——沒改就會照舊架構生 code。

【三類要改的東西】
1. VariableCreate.cs / BuildModel.cs 包裝類別 → 已廢除，改為 Program.cs 的
   OptModel.AddVariables / AddObjective / AddConstraints
2. ExperimentRunner.cs / RunExperiment() → 已廢除，改為 OptExperiment
3. CplexConfig 的 enableLog / exportLP / exportMPS / exportSol → 搬到 ProjectConfig
   （注意：enableLog 舊預設是 true）
4. new OptModel("專案名") 的 runner 用法 → new OptProject(model)，
   那個字串是「專案名」要搬去 ProjectConfig.ProjectName

【額外的前置同步】
規格 §前置 記載：Template 的 Constraint / Objective 已改為「建構子只收自己用得到的
Set 積木與 Parameter 清單，不收整包 Dataload」，但下列文件文字尚未同步，一併改掉：
  $FW/MILP Model/optimfoundation-api-guide.md §4.2 §4.3 §5
  $FW/MILP Model/Foundation Coding/CLAUDE.md
  （另兩份 Template/Constraint、Template/Objective 的 CLAUDE.md 歸 Packet 4）

【驗收】對本包改過的每一份檔跑：
  rg "VariableCreate|BuildModel\.cs|ExperimentRunner|enableLog|exportLP|exportMPS|exportSol" <檔>
預期 0 hit（明確標示為「舊寫法對照」的範例除外，那類要保留但加註「已廢除」）。

【絕對禁止】
- NEVER 改任何 .cs 檔（本包只動 .md）
- NEVER 動 Tier 2/3/4/5 的檔（那是別包）
- 檔案內容與規格描述對不上 → 跳過並回報，NEVER 自行推測
- 寫檔規則：NEVER 欄位對齊（= / : / 註解一律單一空格）；NEVER 產生 UTF-8 BOM

【回報】逐檔列出改了什麼、rg 驗收結果、跳過的檔與原因。
```

---

# Packet 2b — 文件 Tier 2（API 權威，2 份）

**執行位置**：分兩次跑（`AI-Modeling` 與 `OptimFoundation\OptimFoundation`）

```
你是技術文件維護者，任務是更新兩份 API 權威文件。

【唯一權威來源】
specs/2026-08-01-optim-docs-and-projects-migration.md §檔案清單 → Tier 2
以及另外兩份規格（dual-config、runner-symmetry）的 API Design 章節。

【要改的兩份】
  AI-Modeling/CPLEX_API_REFERENCE.md
    §4 CplexConfig（移除 6 個成員 + 新增 ProjectConfig 一節）
    §12 標準執行流程、§14 完整最小範例、§16 Experiment 套件、附錄速查卡
  OptimFoundation/specs/developer-guide.md
    13 章全面對照新 API

【必須順手修正的既有錯誤】
CPLEX_API_REFERENCE.md §4 寫 enableLog 預設 false，
但原始碼 CplexConfig.cs 第 18 行是 true。
新文件的 ProjectConfig.EnableSolverLog 預設必須寫 true。

【驗收】
  rg "enableLog|exportLP|exportMPS|exportSol|LogToConsole|LogFilePath" <兩份檔>
預期只在「已移除成員」的說明段落出現，不得出現在任何建議寫法的範例裡。

【絕對禁止】NEVER 改 .cs、NEVER 動別包的檔、欄位對齊、UTF-8 BOM
【回報】逐節列出改了什麼、驗收結果。
```

---

# Packet 2c — 文件 Tier 3（教學導覽，9 份）

**執行位置**：分三次跑（`AI-Modeling`、`OptimFoundation\OptimFoundation`、`LLMDevFramework`）

```
你是技術文件維護者，任務是更新教學與導覽文件。

【唯一權威來源】
specs/2026-08-01-optim-docs-and-projects-migration.md §檔案清單 → Tier 3

【9 份清單】
  AI-Modeling/README.md
  AI-Modeling/CodeMap.md
  AI-Modeling/ROADMAP.md
  AI-Modeling/tuning/CLAUDE.md
  AI-Modeling/tutorial/ai-modeling-framework-tutorial.md
  AI-Modeling/tutorial/development-workflow.md
  OptimFoundation/specs/cplex-project-dev-spec.md
  OptimFoundation/specs/framework-dev-spec.md
  LLMDevFramework/CodeMap.md

【改什麼】同 Packet 2a 的四類（包裝類別、ExperimentRunner、四個輸出開關、OptModel→OptProject）。
CodeMap 類的檔還要更新 Dependency Graph 與 File Index 的行數。

【驗收】
  rg "VariableCreate|BuildModel\.cs|ExperimentRunner|enableLog|exportLP" <各檔>
預期 0 hit。

【絕對禁止】NEVER 改 .cs、NEVER 動別包的檔、欄位對齊、UTF-8 BOM
【回報】逐檔列出改了什麼、驗收結果。
```

---

# Packet 2d — 文件 Tier 5（歷史 spec 加 frontmatter，6 份）

**執行位置**：分三次跑

```
你是文件維護者，任務只有一件：替 6 份已被取代的舊 spec 加上 frontmatter 標記。

【要改的 6 份】
  AI-Modeling/specs/2026-07-07-tech-doc-story-loop.md
  AI-Modeling/specs/2026-07-21-newcomer-onboarding.md
  AI-Modeling/specs/2026-07-22-tutorial-reference-architecture.md
  OptimFoundation/specs/2026-06-18-experiment-tuning-tracking.md
  OptimFoundation/specs/2026-07-13-optset-basic-objects.md
  LLMDevFramework/specs/2026-07-14-ai-modeling-governance-wave2.md

【怎麼改】在既有 frontmatter 加一行（沒有 frontmatter 就補一個）：
  superseded_by: 2026-08-01-optimfoundation-dual-config.md, 2026-08-01-optimfoundation-runner-symmetry.md, 2026-08-01-optim-docs-and-projects-migration.md

【絕對禁止】
- NEVER 改動檔案內文一個字（只加 frontmatter 那一行）
- NEVER 刪除任何檔案
- 有些檔 status 是 draft（從未執行完成），一樣照加，不要試圖判斷或改寫

【驗收】6 份都有 superseded_by；git diff 顯示每份只多一行。
【回報】列出 6 份的 diff 行數（應各為 +1）。
```

---

# Packet 3 — 更新 dlls/（驗收反直覺，務必看完）

**執行位置**：`...\OptimizationFramework`（跨兩個 repo）　**前置**：1a、1b、2a 都完成

```
你的任務只有一件：把新 build 的框架 DLL 放進 AI-Modeling/dlls/。

【步驟】
1. cd OptimFoundation\OptimFoundation
   dotnet build -c Release          → 確認 0 error
2. 把下列 DLL 從 build 輸出複製到 ..\..\AI-Modeling\dlls\（覆蓋）：
     OptimFoundation.Core.dll
     OptimFoundation.Cplex.dll
     OptimFoundation.Generators.dll
   （ILOG.Concert.dll / ILOG.CPLEX.dll / NLog.dll 不變，不要動）
3. 更新 AI-Modeling\dlls\VERSION.txt：移除 pin 標記，寫上本次 build 日期與內容摘要

【驗收 —— 注意，這包做完 AI-Modeling 應該是「紅的」】
  ✅ dlls\ 三個 DLL 的時間戳 = 本次 build
  ✅ VERSION.txt 已解 pin
  ✅ cd AI-Modeling\Template && dotnet build
       → 預期出現 CS0117（CplexConfig 未包含 enableLog 的定義）
       這是正確狀態，Packet 4 會修

【絕對禁止】
- NEVER 因為看到 CS0117 就去改任何 .cs —— 那是 Packet 4 的工作，
  會與 Packet 4 的 agent 撞車
- NEVER 動 AI-Modeling 底下除了 dlls\ 以外的任何東西

【回報】三個 DLL 的新時間戳、VERSION.txt 內容、Template 的 CS0117 錯誤訊息原文。
```

---

# Packet 4 — 專案遷移（8 個，可平行；每個 agent 一份）

**執行位置**：`...\OptimizationFramework\AI-Modeling`　**前置**：2a、3 都完成

> 下面 `{專案}` 換成：`ClinicVitamin`、`FactorioOptimization`、`GlassFactory`、`HospitalRostering_Generator`、`HospitalRostering_Manual`、`MaxWeightIndependentSet`、`SandwichProduction`、`WeeniesBuns`
> 另外 `Template` 用同一份，檔案只有 `Template/Program.cs`。

```
你是 C# 開發者，任務是把一個練習專案遷移到新的 config / runner API。

【唯一權威來源】
specs/2026-08-01-optim-docs-and-projects-migration.md
  §before / after pattern（Pattern A 與 Pattern B，逐字照做）
  §給執行 agent 的硬規則

【工作範圍】只動這兩個檔：
  Projects/{專案}/Program.cs           ← 改寫
  Projects/{專案}/ExperimentRunner.cs  ← 整檔刪除，內容用 OptExperiment 收進 Program.cs
以及該專案的 CLAUDE.md（若有，同步描述）

  例外：HospitalRostering_Manual 沒有 Program.cs，改的是 HospitalRosteringProblem.cs
        MaxWeightIndependentSet 的 code 在 Project\ 子資料夾底下

【驗收 —— 基準值是唯一依據】
先讀 baseline/BASELINE.md 找到 {專案} 那一列，以及 baseline/{專案}.txt 的原始輸出。
遷移後跑：
  cd Projects\{專案}
  dotnet build     → 0 error
  dotnet run       → 比對三項：
                      ① Status 仍為 Optimal
                      ② 目標值與 BASELINE.md 一致（浮點相對差 ≤ 1e-6）
                      ③ 各 [Constraint_*] 條數與 baseline/{專案}.txt 一致

【絕對禁止 —— 這幾條最重要】
- NEVER 改任何模型的數學內容。本任務只做 config / runner 遷移。
  目標值變了 = bug，不是預期。
- NEVER 為了讓驗收通過而調整容差、改基準值、或修改 BASELINE.md
- 目標值對不上 → 停下回報，NEVER 自行「喬」成一致
- 檔案內容與 before pattern 不符 → 跳過並回報，NEVER 自行推測
- build 修 5 次仍失敗 → 停下回報
- NEVER 動其他專案的檔案

【三個容易做錯的地方】
1. 舊的 new OptModel("{專案}") 那個字串是「專案名」→ 搬去 ProjectConfig.ProjectName。
   新的 OptModel 名字是「模型名」，用於實驗 label，語意不同。
2. 舊的 AddModel(e => new BuildModel(...).Build()) 要拆成兩個：
   原 BuildModel.Build() 內第一行 new ObjectiveFunction(...).Build() → AddObjective
   其餘的 new Constraint_*(...).Build() → AddConstraints
3. 舊 ExperimentRunner 裡的 enableLog = false 不用搬 —— OptExperiment 預設就是 log OFF。
   每個 variant 各自 OptData.Load 的舊寫法改為共用一份 data。

【回報】改了哪些檔、刪了哪個檔、build 結果、遷移前後的 Status / 目標值 / 限制式條數對照表。
```
