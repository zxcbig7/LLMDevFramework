# Foundation Coding — Phase 2 轉譯實作規範

<system_context>
三階段的第二階段：把已確認的 `Model/<Project>_Model.md` **逐條機械轉譯**成 OptimFoundation CPLEX C# 專案。
天條（LHS/RHS 鐵則、禁止 Hardcode、命名對應）見 `../CLAUDE.md`；本檔放結構與 API 細節。
API 簽名的權威來源：`$OPT/AI-Modeling/CPLEX_API_REFERENCE.md` + `$OPT/.../specs/developer-guide.md`。
</system_context>

<critical_notes>
- MUST 專案建在 `$OPT/AI-Modeling/Projects/<Project>/`，DLL 一律參考 `$OPT/AI-Modeling/dlls/`（HintPath `..\..\dlls\Xxx.dll`）—— Why: DLL 唯一來源天條，路徑亂掉 build 就靠運氣
- MUST `Parameter\` 資料夾必須存在；Set 與 Parameter 各自從 `IDataSource` 讀進來——Set 預設 `ITEM.Load(source)`（內部即 `source.LoadSet("Set_Item")`），NEVER 一律用 `LoadFrom(parameter_Xxx.Select(...).Distinct())` 從參數反推 —— Why: 反推出來的 set 只有「這批參數用到過」的成員，題目允許但資料沒出現的維度會靜默消失，變數少建、約束少建，解看起來正常但不是原題。真的沒有來源資料的 set 才用 `LoadFrom` 衍生（逃生口）
- MUST 建模方式預設 source generator + Fluent `OptModel`；需逐行掌控引擎生命週期才退回手寫 `: VariableBase` / `XxxProblem.Execute()` —— 範例分別見 `Projects/HospitalRostering_Generator` 與 `Projects/HospitalRostering_Manual`
  - paved path：Set 積木 `[OptSet<T>]` + 泛型引用 `[OptVar<Set_X>]` / `[OptParam<Set_X>]`（property 名/型別從積木自動抓）；字串式 `[OptVar("Date:DateTime")]` 為逃生口。完整設計見 `$OPT/OptimFoundation/OptimFoundation/specs/2026-07-13-optset-basic-objects.md`
  - NEVER 寫裸 `[OptSet]` —— ALWAYS 顯式 `[OptSet<string>]` —— Why: 裸版隱含 string，宣告處看不出元素型別；codegen 兩者相同，顯式零成本
- MUST 參數讀取先 LINQ 存局部變數再傳入 `AddLHS` / `AddRHS`，NEVER 把 LINQ 內嵌在呼叫裡 —— Why: 可 debug、可驗值，內嵌讀不出中間值
- MUST 變數建立與模型組裝寫成 `Program.cs` 的 local function（`CreateVariables` / `BuildModel`），NEVER 另外開 `VariableCreate.cs` / `BuildModel.cs` 包裝類別 —— Why: 那兩層只有轉呼叫、沒有邏輯，拆出去之後「這專案怎麼組起來」要開三個檔才看得完；寫在 Program.cs 一眼看完，且 solve 與 experiment 天然共用同一份
- MUST `BuildModel()` 只呼叫各 `Constraint_Xxx.Build()` 與 `ObjectiveFunction`，NEVER 在裡面直接寫 `AddLHS` / `AddRHS`
- MUST 目標式排在所有 constraint 之前建 —— Why: soft constraint 的 penalty 是掛進既有目標式的，順序反了會掛空
- MUST Variable class 只放 properties、不寫 constructor（框架用 reflection 組 key）
- MUST build 失敗走 fix loop：擷取 compiler error → 修正 → 重 build，**至多 5 次**；仍失敗 → 停下回報，NEVER 硬掰
</critical_notes>

<paved_path>
## 專案結構（五資料夾，逐條對應 Model.md）

```text
Projects/<Project>/
├── <Project>.csproj # DLL HintPath ..\..\dlls\
├── Program.cs # 唯一進入點：solve / experiment 雙模式 + 全部層級組裝（含參數掃描）
├── Model/ # <Project>_Model.md + Glossary.md（Phase 1 產物）
├── Set/ # Set 積木（[OptSet<T>]）+ Dataload（顯式 ctor 逐行讀檔；換 CSV/Oracle/記憶體只換 IDataSource）
├── Parameter/ # Parameter_Xxx.cs（[OptParam] + [OptDim<Set_X>] 生成，QTY 欄位）
├── Variable/ # VariableB_/X_/I_Xxx.cs
├── Objective/ # ObjectiveFunction.cs
└── Constraint/ # Constraint_Xxx.cs
```

Namespace = `<Project>.<資料夾名>`（根目錄 = `<Project>`）。

## Program.cs 骨架（唯一組裝點，solve / experiment 雙模式）

```csharp
if (args.Contains("experiment")) { RunExperiment(); return; }

var dataload = OptData.Load(() => new Dataload());   // 唯一建構入口，跳過就沒有資料驗證
using (var model = new OptModel("<Project>")
    .UseConfig(() => NewConfig(timeLimit: 300, verbose: true))
    .AddVariables(engine => CreateVariables(dataload, engine))
    .AddModel(engine => BuildModel(dataload, engine))
    .OnSolved(engine => dataload.WriteToCSV(engine)))
{
    bool ok = model.Execute();
}

return;

static void CreateVariables(Dataload data, OptEngine engine)
{
    engine.BuildBVs<VariableB_Assign>(data.EMPLOYEE, data.DATE);
    Logging.Info($"變數建立完成：{engine.varCount}");
}

static void BuildModel(Dataload data, OptEngine engine)
{
    new ObjectiveFunction(data, engine).Build();     // MUST 先建
    new Constraint_MaxWorkDays(data, engine).Build();
}

static CplexConfig NewConfig(double timeLimit, bool verbose) => new CplexConfig
{
    epGap = 1e-4,
    timeLimit = timeLimit,
    workThreads = 8,
    enableLog = verbose,
    exportSol = verbose,
    exportLP = verbose,
};
```

`CreateVariables` / `BuildModel` 是 local function，solve 與 experiment 兩模式共用同一份建模碼——不會出現「掃參數時跑到另一個模型」。
端到端完整教學（含 Set/Parameter/Dataload/Constraint 逐節寫法、實驗、驗收、API 速查）見 [`../optimfoundation-api-guide.md`](../optimfoundation-api-guide.md)。

## 轉譯順序

1. Parameter（Model.md Parameters 表逐列）→ 2. Dataload（數值保真）→ 3. Variable → 4. Constraint 逐條（一條/一組邏輯相關 = 一個 `Constraint_Xxx.cs`）→ 5. ObjectiveFunction → 6. Program.cs（`CreateVariables` + `BuildModel` + `RunExperiment`）→ 7. `dotnet build` → fix loop ≤5 → `dotnet run` → **解驗證協定**（見 verification）
</paved_path>

<patterns>
## Pool API（限制式）

```csharp
foreach (var e in dataload.Employees)
{
    foreach (var d in dataload.Dates)
        engine.AddLHS(1.0, new VariableB_Assign { EMPLOYEE = e, DATE = d });
    var maxDays = dataload.parameter_MaxWorkDays.FirstOrDefault(p => p.EMPLOYEE == e)?.QTY ?? 0.0;
    engine.AddRHS(maxDays);
    engine.CreateLessEqual($"MaxWorkDays@{e}");
    ConstraintCount++;
}
```

- 約束名格式：`ConstraintName@index1@index2`
- RHS 含變數（移項需求）→ `AddRHS(coef, variable)`，但 Model 原式怎麼寫就怎麼放，NEVER 自行移項

## 取解 API（存在的才用）

```csharp
double obj = engine.GetObjectiveValue();
var sol = engine.GetSetVarValues<VariableX_Production>(); // {"VariableX_Production@Regular": 60.0}
double v = engine.GetVariableValue("VariableB_Assign@E1@2026-01-01"); // DateTime 格式 @yyyy-MM-dd
FolderDir.Solution.CreateFolder(); // ★ WriteSolution 前必呼叫
CsvCtrl.WriteSolution<VariableX_Production>(engine, "<Project>", "USER");
```

## 禁止使用（Foundation 不存在這些方法）

```csharp
// ✗ engine.GetVarSol(...) → 不存在
// ✗ engine.GetSetVarSol<T>() → 不存在
// ✗ CsvCtrl.SaveToCSV<T>(...) → 不存在（正確：WriteSolution）
```

簽名有疑慮 → 查 `CPLEX_API_REFERENCE.md`，NEVER 憑記憶發明 API。

## good/bad：參數讀取

✅ Good
```csharp
var profit = dataload.parameter_Profit.FirstOrDefault(p => p.GLASS_TYPE == g)?.QTY ?? 0.0;
engine.AddLHS(profit, new VariableX_Production { GLASS_TYPE = g });
```

❌ Bad
```csharp
engine.AddLHS(dataload.parameter_Profit.First(p => p.GLASS_TYPE == g).QTY, new VariableX_Production { GLASS_TYPE = g }); // LINQ 內嵌
engine.AddLHS(300.0, new VariableX_Production { GLASS_TYPE = g }); // 裸數字
```
</patterns>

<verification>
## 解驗證協定（`dotnet run` 後必跑，全過才算「會動」）

MUST 依序四步，任一不過 → 停下回報，NEVER 宣稱完成：

1. **Status 三分診斷**
   - `Optimal` → 進第 2 步
   - `Infeasible` → 回報並走 Tuning 的 IIS 流程（`Foundation Tuning/CLAUDE.md`）；先自查 big-M 是否太小、有無互斥硬約束
   - `Unbounded` → 某方向漏了界；查該變數 UB 或漏掉的上限約束
2. **可行性代回**：取解值代回**每一條** constraint，確認 LHS op RHS 成立（含 soft/big-M）—— Why: solver 回 Optimal 只保證它解的模型可行，不保證那模型 = 題目
3. **單位一致性**：目標值與關鍵變數的單位跟題目一致（利潤=錢、產量=件、工時=時），量級合理
4. **LP bound sanity**：目標值落在 LP relaxation bound 對的一側（max 問題：整數解 ≤ LP bound；min 問題：整數解 ≥ LP bound）；差太離譜 → 疑 big-M / 係數錯

✅ Good：四步都過 + 目標值對照 Model.md 手算小例
❌ Bad：看到 `Status == Optimal` 就回報「解出來了」（沒代回、沒對單位）
</verification>

<common_tasks>
- 新專案起手 → 依 `$OPT/AI-Modeling/claudemdTemplate/` 各資料夾模板 + 參考 HospitalRostering_Generator
- csproj DLL 區塊 → 抄範例專案的 `<ItemGroup>`（五個 Reference：ILOG.Concert / ILOG.CPLEX / NLog / OptimFoundation.Core / OptimFoundation.Cplex）
- build 錯 CS0246（找不到型別）→ 先檢查 HintPath 相對層數（Projects 下是 `..\..\dlls\`）
- 驗解 → 目標值 + 關鍵變數值對照 Model.md 手算小例；`exportSol = true` 留 .sol 檔
</common_tasks>

<fatal_implications>
- NEVER 自行詮釋 Model.md（歧義 → 回 Phase 1）
- NEVER 裸數字進 Constraint / Objective
- NEVER 用別的 DLL 路徑
- NEVER fix loop 超過 5 次還繼續硬修
- NEVER 改 OptimFoundation 框架本體
</fatal_implications>
