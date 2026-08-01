# 規格：AI-Modeling 治理第二波 —— API 寫法唯一化 + 舊規範清理

status: approved（缺陷經 Explore 盤點驗證，D15/D16 刪除決策由 Vic 於 2026-07-14 拍板）
executor: 待派（照第一波模式執行，每 phase 結尾派 verifier）
scope: `$OPT/AI-Modeling`（主）+ 條件式檢查 `$FW/MILP Model`（$OPT = `C:/Users/zxcbi/Desktop/Projects/OptimizationFramework`、$FW = LLMDevFramework repo 根）
前情: 第一波 = `specs/2026-07-13-optim-ai-spec-consolidation.md`，4 phase 已於 2026-07-14 執行完畢（verifier PASS）。本波處理第一波**未納管而存活**的殘洞。

> **API 定版（Vic 2026-07-15 指定，覆蓋本規格與第一波 D5 的語法描述）**：
> 權威範例 = `$OPT/OptimFoundation/OptimFoundation/Templates/FJSP_BASIC_BRICK`（實作已 build 過、Generated/ 有 generator 輸出）。
> 治理文件教的語法以該 template 實際寫法為準（見 D13 修訂版）；第一波 D5 記載的 `[OptVar<Set_A, Set_B>]` 多參數泛型仍存在於框架，但**降級為逃生口**，template 未使用。
> **資料夾結構不跟 template**（Vic 2026-07-15 拍板）：AI-Modeling 專案維持 D3 扁平六夾（Model/Set/Parameter/Variable/Constraint/Objective）；template 的 SetClass/ParameterClass/VariableClass/Data 命名是 OptimFoundation repo 自己的慣例，不進 AI-Modeling 治理文件。

## 定位（本波一切取捨的準繩，寫進 AGENTS.md 檔首）

> AI-Modeling 唯一精神：**把數學模型機械轉譯成 OptimFoundation code ＋ 建實驗**。
> 非框架開發區、非教學區、非 API 寫法實驗場。任何文件教的寫法只能有一種 paved path。

---

## 一、缺陷清單（2026-07-14 Explore 盤點驗證）

| # | 缺陷 | 證據 |
|---|---|---|
| B10 | **automated/Prompts 教第 4 種 API 寫法，從未隨 CLAUDE.md 改版同步**：`Param_` 前綴（非 `Parameter_`）、`VariableY_` 前綴（無 B/X/I 之分）、property ALL_CAPS（`PRODUCT_INDEX`）、namespace 硬編碼 `Model`、手寫 `InitClassBySets` 不用 generator。同資料夾 `automated/CLAUDE.md:78` 已用 `VariableB_` + 泛型積木——同夾自相矛盾。這是**實際餵給 LLM 的 prompt 本體**，不改則 automated 路線量產的 code 持續分裂 | `Prompts/05_ParamCode.md:22,40`、`06_VarCode.md:24`、`07_DataloadCode.md:13`、`10_VarCreateCode.md:41` |
| B11 | `claudemdTemplate/`（AGENTS.md 宣稱的資料夾模板單一來源）**缺 OptSet 泛型寫法**（落後 `CPLEX_API_REFERENCE.md` §7.5），且 `Variable/CLAUDE.md:41` 教假 API `BuildXVs`（真實只有 `BuildBVs/BuildCVs/BuildIVs`）——「單一來源」名不副實 | 兩檔實測 |
| B12 | **「建實驗」無單一權威**：ExperimentRunner 樣板近似重複 3 處（`claudemdTemplate/Experiment/`、`interactive/phase-3-tuning.md:54-67`、`README.md`）；資料集切換只有 phase-3 一行帶過；automated 路線缺 `Prompts/15_ModelTuning.md`（ROADMAP 待辦懸置） | 三檔實測 |
| B13 | 零散 stale：`Template_CPLEX/CLAUDE.md:25-50` 結構圖缺 `Model/`（畫五夾，磁碟實際六夾）；`interactive/README.md:33`、`interactive/phase-2-coding.md:20` 標題「五資料夾」但內容列六項；`Projects/FactorioOptimization/CLAUDE.md` 描述不存在的 `FactorioOptimizationProblem.Execute()`（實際是 `OptModel` Fluent composition root）；`Projects/MaxWeightIndependentSet/Model/*_Model.md` 是 19 行驗證報告而非模型文件；MaxWeightIndependentSet 的 code 用 `VariableY_Select`/`Param_Weight` 違反命名天條，且結構仍是 Model/+Project/ 混合式（第一波 2-8 未完成的殘項） | 各檔實測 |
| B14 | 四份舊 draft spec 未清理（第一波範圍註刻意排除）：`automated/specs/2026-05-20-*`（Web API/RAG 藍圖，已放棄路線）、`automated/specs/2026-06-21-*`（AML 術語滿篇）、`specs/2026-06-22-dual-architecture-CodeMap.md` 與 `-tutorial.md`（引用從未存在的 `OptimModeling.Generators`）——LLM 與人都會誤讀 | 各檔實測 + ROADMAP.md:47 |
| B15 | `Projects/HospitalRosteringProblem_new/` 舊 Registry 架構前身仍在 repo（ROADMAP.md:105 早列待刪候選） | 目錄實測 |

## 二、決策（Vic 拍板 2026-07-14）

| # | 決策 | 內容 |
|---|---|---|
| D13 | **API 寫法唯一 paved path = brick API（FJSP_BASIC_BRICK 版）** | 見下方「D13 定版語法」。治理文件 NEVER 出現 `Param_` 前綴 / `VariableY_` / ALL_CAPS property / `BuildXVs`；constraint 建構檔名 = `Constraint/BuildModel.cs`（對齊全部真實專案，廢 `BuildConstraints` 稱呼與檔名） |
| D14 | **建實驗權威二分** | 建流程樣板權威 = `claudemdTemplate/Experiment/CLAUDE.md`（擴充：含資料集切換協定——只改 Dataload 載入呼叫，五夾零改動）；solver 旋鈕權威 = `tuning/CLAUDE.md`（第一波 D7 不變）；`interactive/phase-3-tuning.md` 與 `README.md` 的樣板段改一行指向；補產 `Prompts/15_ModelTuning.md`（以 `tuning/CLAUDE.md` 為源改寫成 prompt 模板） |
| D15 | **四份舊 draft spec 直接刪除**（翻案第一波範圍註「不竄改歷史」——Vic：舊規範造成混亂優先於保史，git 歷史即封存） | 刪 B14 列的四檔；`specs/2026-07-07-*` 現行有效保留；`tutorial/` 維持排除不動 |
| D16 | **HospitalRosteringProblem_new 整案刪除** | 刪整資料夾；ROADMAP 待辦同步；第一波 B8② 對它的修復如未執行即作廢 |

### D13 定版語法（權威範例 = FJSP_BASIC_BRICK，2026-07-15 實讀確認）

| 元件 | 唯一 paved path | 範例出處 |
|---|---|---|
| Set | `[OptSet<T>]` partial class，元素型別 MUST 顯式（string 寫 `[OptSet<string>]`；裸 `[OptSet]` 仍合法但非預設）；generator 補 `: SetBase<T>` | `SetClass/Sets.cs` |
| Var/Param 維度 | 光桿 `[OptVar]`/`[OptParam]` + 逐維 `[OptDim<Set_X>("Name")]`（attribute 順序 = key 順序）；同 set 多維度用角色名區分（`LotA`/`LotB`）；scalar = 光桿 `[OptVar]` 零維，key 無 `@` | `VariableClass/VariableB_Assign.cs`、`VariableB_Precede.cs`、`VariableX_Makespan.cs`、`ParameterClass/Parameter_ProcessTime.cs` |
| 變數建立 | 統一 `BuildVars<T>(積木...)`，型別由類名前綴 B_/X_/I_ 決定；直接傳 Set 積木（SetBase 是 IEnumerable） | `VariableClass/VariableCreate.cs` |
| 資料層 | `IDataSource` 抽象（`CsvDataSource`/`DbDataSource`）：`ReadParameters<T>()` + 積木 `LoadCsv("Set_X")` / `LoadFrom(source.ReadSet("X"))`；Dataload 保留 List 視圖（`XSet = X.ToList()`）供索引型 constraint；載入後 fail-fast 驗證（set 覆蓋 parameter 值域）；BigM 由數據推導 NEVER 寫死 | `Data/Dataload.cs` |
| Constraint/Objective | `AddLHS(coef, new VariableX_Y { Dim = val })` / `AddRHS` / `CreateGreatEqual($"{ConstraintName}@{key}")` / `CreateMinimize`；繼承 `ConstraintBase` | `Constraint/Constraint_MakespanDef.cs`、`ObjectiveFunction.cs` |
| 組裝與求解 | `OptModel(name).UseConfig(() => new CplexConfig{...}).AddModel(...).OnSolved(...)` Fluent + `Execute()`；infeasible → `GetConflictConstraints()`（IIS） | `Program.cs` |
| 解回讀 / 輸出 | `GetSetVarValues<T>()` 批量取解；`CsvSolutionSink.WriteSolution<T>(engine)` | `Data/Dataload.cs`、`Program.cs` |
| 實驗 | `Experiment(name, desc)` + `exp.AddTrial(Trial.Capture(engine, label, () => engine.Solve()))` + `exp.Save()` → `Experiments/*.csv/.json/-trajectory.csv` | `Program.cs` `RunCrossExperiment` |

逃生口（治理文件僅可標注為逃生口，prompt 模板不教）：多參數泛型 `[OptVar<T1..T6>]`/`[OptParam<T1..T6>]`、字串式 `[OptVar("Date:DateTime")]`、底層 `BuildBVs/BuildCVs/BuildIVs`＋List 視圖直建。

## 三、執行計畫（每 phase 結尾派 verifier fresh-context 驗收）

### Phase A — 刪除與零散 stale（低風險，先做）

0. **前置：commit 第一波成果**——AI-Modeling repo 現有 5 檔 modified 未 commit（AGENTS/CLAUDE/CPLEX_API_REFERENCE/CodeMap/Projects-CLAUDE），MUST 先 commit 才動刪除 —— Why: 刪除的可回復性靠 git，未 commit 就刪等於裸奔
1. D15：刪四份 draft spec；`automated/specs/` 若因此變空夾一併刪
2. D16：刪 `Projects/HospitalRosteringProblem_new/` 整夾
3. `ROADMAP.md` 同步：Spec 索引移除四檔、待刪待辦銷項、決策史記一筆（含 D15 翻案理由）
4. B13 文件修正：`Template_CPLEX/CLAUDE.md` 結構圖補 `Model/`；interactive 兩處「五資料夾」→「六資料夾」；`FactorioOptimization/CLAUDE.md` 執行流程對照 `Program.cs` 現實重寫

驗收：glob 確認五個刪除標的不存在；grep AI-Modeling 治理文件（排除 tutorial/、packages/、Generated/）無「五資料夾」；FactorioOptimization/CLAUDE.md 描述與 Program.cs 實碼一致；git log 有前置 commit。

### Phase B — API 寫法唯一化（D13，本波核心）

0. **API 文件鏈先定版**：驗 OptimFoundation `specs/developer-guide.md`（源頭）是否完整記載「D13 定版語法」全表（OptSet/OptDim/BuildVars/IDataSource/GetSetVarValues/Experiment）——缺則先補源頭；再同步 `CPLEX_API_REFERENCE.md` 鏡像（D8 鏈，檔頭更新同步日期）。§7.5 若仍以多參數泛型為主寫法 → 改 OptDim 為主、泛型降逃生口
1. `AGENTS.md`：檔首加定位句（見上方「定位」）+ D13 天條入天條區
2. `automated/Prompts/05/06/07/10/11` 內容重寫對齊「D13 定版語法」：`Parameter_` / `VariableB_`,`VariableX_`,`VariableI_` / property PascalCase / 光桿 attribute + `[OptDim<Set_X>("Name")]` + `BuildVars<T>`（廢手寫 `InitClassBySets` 教學）/ namespace 對齊現行專案慣例；`11_BuildConstraintsCode.md` 檔名與內文改 `BuildModel`
3. `claudemdTemplate/` 全部子模板同步「D13 定版語法」；修 `BuildXVs` 筆誤
4. `Template_CPLEX/CLAUDE.md` 語法範例段：字串式範例改 OptDim 寫法，或就地標注「逃生口（遷移用）」

驗收：grep AI-Modeling 治理文件 + Prompts/（同上排除）無 `Param_`、`VariableY_`、`BuildXVs`、`BuildConstraints`、`PRODUCT_INDEX`；claudemdTemplate、CPLEX_API_REFERENCE §7.5、developer-guide 三處 attribute 教法一致 = 「D13 定版語法」（verifier 對照本規格該表逐列打分）。

### Phase C — 建實驗權威（D14）+ MaxWeightIndependentSet 收尾

1. `claudemdTemplate/Experiment/CLAUDE.md` 擴成唯一樣板權威（ExperimentRunner 全文 + 資料集切換協定）；`interactive/phase-3-tuning.md` 與 `README.md` 樣板段改一行指向
2. 產 `Prompts/15_ModelTuning.md`（源 = `tuning/CLAUDE.md`，銷 ROADMAP 待辦）
3. MaxWeightIndependentSet 遷移扁平六夾 + 改名 `VariableY_Select`→`VariableB_Select`、`Param_Weight`→`Parameter_Weight`（純命名/結構重構）；重建真正的 `*_Model.md`（SET/PARAM/VAR/CONSTRAINT/OBJ 五段，驗證報告內容併入或另存 `experiments/`）；MUST `dotnet build` 綠

驗收：全部 8 個 Projects + Template_CPLEX `dotnet build` 全綠；ExperimentRunner 樣板全文只在一處、其餘 ≤1 行指向；grep 含 `.cs` 無 `VariableY_`、`Param_` 前綴；Projects/ 結構型態 = 扁平六夾唯一。

### Phase D — $FW/MILP Model 條件式同步（做完 A–C 才判斷）

1. grep `$FW/MILP Model` + `$FW/workflow skill` 是否有 `Param_`/`VariableY_`/`BuildXVs`/字串式 attribute 教學殘留——有才改，沒有即結案
2. 有改動 → MUST 跑 `/harness-eval MILP Model` 驗無 regression + 同步 root `CodeMap.md` File Index

## 四、通用鐵則（承第一波，全程適用）

- NEVER 動數學語意：全部是文件 / 命名 / 結構重構，不改任何模型的移項、符號、數值
- 每 phase 結束 MUST 派 verifier fresh-context 驗收後才進下一 phase
- 檔案改動遵守 $FW 鐵則：無欄位對齊、無 BOM、CLAUDE.md ≤ 200 行
- 計數類事實（專案數、行數）MUST 執行當下實測，不抄本規格
- 刪除只走 `git rm` / 檔案系統刪除後 commit，NEVER 額外做 zip 備份（git 即封存）
