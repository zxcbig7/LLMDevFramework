# Foundation Coding — Phase 2 轉譯實作規範

<system_context>
把已確認的 `Model/<Project>_Model.md` 逐條機械轉譯成 OptimFoundation CPLEX C# 專案。天條見 `../CLAUDE.md`；完整教學版與 API 黑名單見 `../optimfoundation-api-guide.md`；API 簽名以 `$OPT/AI-Modeling/CPLEX_API_REFERENCE.md` 為準。
</system_context>

<critical_notes>

- 專案建在 `$OPT/AI-Modeling/Projects/<Project>/`，DLL HintPath 一律 `..\..\dlls\` —— NEVER `ProjectReference`、NEVER NuGet、NEVER CPLEX Studio 安裝路徑。限定例外：維護 `$OPT/OptimFoundation/OptimFoundation/Templates/*` 時保留 framework source / generator `ProjectReference` 與 solver `$(CplexDir)` HintPath，因其屬框架 solution 的同步 build surface；不得把此例外帶回 AI-Modeling 專案。
- **結構固定八個資料夾，NEVER 增減**。一個型別一個 `.cs` 檔，檔名 = 類別名。
- **`Model/` 只放單一 `<Project>_Model.md`**（含 Terminology Mapping Table）；NEVER 另建 `Glossary.md`。`Dataload.cs` 放 `Data/`，與它讀的 CSV 同層。
- **全專案單一 namespace `<Project>`**，block 寫法 `namespace X { }`；每個型別 `sealed` 且有 `<summary>`。
- **只用 generator**（`[OptSet<T>]` / `[OptParam]` / `[OptVar]` + `[OptDim]`）—— NEVER 手寫 `: VariableBase` / `: ParameterBase`；Parameter 只用 object initializer 建構，NEVER 用位置式 ctor。
- **模型組裝只在 `Program.cs`**，也只有它知道 `Dataload`。Objective / Constraint 建構子逐項列出實際依賴，型別用框架實際型別（`Set_Item` / `List<Parameter_X>` / scalar），NEVER 用 `IReadOnlyList<int>` 這種退化型別。`Solution/<Project>Solution.cs` 是唯一可收整包 `Dataload` 的地方。
- **`OptEngine` 只從 `Build(OptEngine engine)` 進來**，NEVER 進建構子。
- **Dataload 以「input CSV 已就位」為前提**：`Dataload(IDataSource)` 只允許 `Load(source, name)` 與 `LoadParam<T>(name)` 兩種句子。不規則來源（矩陣、原始報表）用選用的 `Dataload(string rawFile)` + `Export()` 先攤平產 CSV。
- 建立變數只用 `engine.BuildVars<T>(sets...)`；界限一律寫成 constraint。
- 出池只用單參數 `CreateXxx(name)`；右側用 `AddRHS(常數)`。
- 目標式 MUST 先於 constraints，且是 Model.md `OBJ` 段的逐項轉譯；沒有 OBJ 段 → 停止，退回 Phase 1。
- 參數查詢先存局部變數，NEVER 內嵌 LINQ 到 `AddLHS` / `AddRHS`。
- build fix loop 最多 5 次；仍失敗停下回報。

</critical_notes>

<paved_path>

## 專案結構

```text
Projects/<Project>/
├── <Project>.csproj      HintPath ..\..\dlls\ + Analyzer + Data\**\*.csv copy
├── Program.cs            三態 CLI（import / exp / 預設）+ 材料 → 模型 → 環境
├── Model/                <Project>_Model.md（含術語表；唯一模型文件）
├── Set/                  Set_*.cs（一顆一檔）
├── Parameter/            Parameter_*.cs
├── Variable/             Variable[B|X|I]_*.cs
├── Objective/            ObjectiveFunction.cs
├── Constraint/           Constraint_*.cs（一條一檔）
├── Solution/             <Project>Solution.cs
└── Data/                 Dataload.cs + Set_*.csv + Parameter_*.csv + raw/（選用）
```

## Program.cs — 唯一組裝點

三態分派在最前，其餘平坦分三段：材料 → 模型 → 環境。

```csharp
// 模式 1：import（只有不規則來源才需要）
if (args.Length >= 2 && args[0] == "import")
{
    OptData.Load(() => new Dataload(args[1])).Export();
    return 0;
}

bool isExperiment = args.Any(arg => string.Equals(arg, "exp", StringComparison.OrdinalIgnoreCase));
if (isExperiment)
    Logging.SetLogFileName("<Project>_exp"); // MUST 在第一次寫入前

// ── 1. 材料 ──
var data = OptData.Load(() => new Dataload());
double penalty = data.parameter_ShortagePenalty.Single().QTY;
var projectConfig = new ProjectConfig { ProjectName = "<Project>", EnableSolverLog = true, ExportLP = true };
var solverConfig = new CplexConfig { epGap = 1e-4, timeLimit = 300, workThreads = 8 };

// ── 2. 模型 ──
var model = new OptModel("Canonical")
    .AddVariables(engine => engine.BuildVars<VariableB_Assign>(data.set_Employee, data.set_Date))
    .AddVariables(engine => engine.BuildVars<VariableX_Shortage>(data.set_Date))
    .AddObjective(engine => new ObjectiveFunction(data.set_Date, penalty).Build(engine))
    .AddConstraints(engine => new Constraint_MaxWorkDays(data.set_Employee, data.set_Date, data.parameter_MaxWorkDays).Build(engine))
    .AddConstraints(engine => new Constraint_Coverage(data.set_Date, data.parameter_Demand).Build(engine));

// ── 3. 環境 ──
// 模式 2：exp → OptExperiment（略）；模式 3：預設 → OptProject
using var project = new OptProject(model)
    .UseConfig(() => projectConfig)
    .UseConfig(() => solverConfig)
    .OnSolved(engine => <Project>Solution.ReadAndValidate(engine, data).Print());

return project.Execute() ? 0 : 1;
```

分工：

- `ProjectConfig` = 專案名、retention、solver log、LP/MPS/Sol 輸出與 `DataId` / `UserId`。
- `CplexConfig` = solver 旋鈕。
- `OptModel` = 可重用模型定義，canonical 一律命名 `"Canonical"`。
- `OptProject` = 一次正式求解；只有 runner 能掛 `OnSolved`。
- `OptExperiment` = model × config；命名 `<Project>-tuning-r<N>`，每輪遞增（同名是 append）。

## 轉譯順序

1. Set / Parameter（含結構常數）。
2. `Data/Dataload.cs`（`IDataSource` ctor；不規則來源再加 import ctor + `Export()`）。
3. Variable。
4. Constraint 逐條。
5. ObjectiveFunction。
6. `Program.cs`：三態分派 + 材料 → 模型 → 環境。
7. `Solution/<Project>Solution.cs`。
8. 備妥 `Data/*.csv`（不規則來源先 `dotnet run -- import raw/<file>`）。
9. build / fix loop ≤ 5 → `dotnet run` → 解驗證協定。

</paved_path>

<patterns>

## Constraint 顯式依賴 + Build(engine)

```csharp
public sealed class Constraint_MaxWorkDays : ConstraintBase
{
    private readonly Set_Employee employees;
    private readonly Set_Date dates;
    private readonly List<Parameter_MaxWorkDays> maxWorkDays;

    public Constraint_MaxWorkDays(Set_Employee employees, Set_Date dates, List<Parameter_MaxWorkDays> maxWorkDays)
    {
        this.employees = employees;
        this.dates = dates;
        this.maxWorkDays = maxWorkDays;
    }

    public void Build(OptEngine engine)
    {
        foreach (var employee in employees)
        {
            foreach (var date in dates)
                engine.AddLHS(1.0, new VariableB_Assign { Employee = employee, Date = date });

            var max = maxWorkDays.FirstOrDefault(p => p.Employee == employee)?.QTY ?? 0.0;
            engine.AddRHS(max);
            engine.CreateLessEqual($"{ConstraintName}@{employee}");
        }
    }
}
```

Model 左邊進 `AddLHS`，右邊進 `AddRHS`；比較方向直接對應 `CreateGreatEqual` / `CreateLessEqual` / `CreateEqual`。NEVER 自行移項。

`ConstraintCount` 已 obsolete，建立群組與數量由 `EngineBase` 自動統計，NEVER 自印。soft variant（`CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft`）只屬 Phase 3 具名 experiment variant。

## Dataload：只讀已就位的 CSV

```csharp
// Data/Dataload.cs
public Dataload() : this(new CsvDataSource()) { }

public Dataload(IDataSource source)
{
    set_Employee.Load(source, "Set_Employee");
    set_Date.Load(source, "Set_Date");
    parameter_MaxWorkDays = source.LoadParam<Parameter_MaxWorkDays>("Parameter_MaxWorkDays");
}
```

白名單外的一切（迴圈、`Random`、日期運算、補值、`if`）只能出現在 `Dataload(string rawFile)` 的 import ctor，且必須先 `Export()` 成 CSV 才進求解。

## 變數界限走 constraint

`BuildVars<T>` 依前綴推型並固定界限：`VariableB_` 為 `[0,1]`、`VariableX_` / `VariableI_` 為 `[0, 1E100]`。
Model.md 的 `x ≤ Capacity` 就建一條 `Constraint_Capacity`，NEVER 用 `BuildCVs(lb, ub, ...)` 把界限藏進參數。

## 取解與驗證

```csharp
var assign = engine.GetSetVarValues<VariableB_Assign>();
double value = engine.GetVariableValue("VariableB_Assign@E1@2026-01-01");
// ValidateRules：逐條把解代回 Model.md 的限制式，不成立就 throw
FolderDir.Solution.CreateFolder();
CsvCtrl.WriteSolution<VariableB_Assign>(engine, "<Project>", "SYSTEM");
```

不存在：`GetVarSol`、`GetSetVarSol<T>`、`SaveToCSV<T>`。已禁用：`BuildBVs` / `BuildCVs` / `BuildIVs`、`CreateXxx(rhs, name)`、`LoadCsv`、`LoadInline`、位置式 Parameter ctor。完整黑名單見 `../optimfoundation-api-guide.md` §9。

</patterns>

<verification>

## 解驗證協定

1. Status：`Optimal` 進下一步；`Infeasible` MUST 讀 `bin/Debug/net8.0/IISs/*.ilp`；`Unbounded` 查漏掉的上限 constraint。
2. 在 `Solution/<Project>Solution.cs` 的 `ValidateRules` 把解代回每條 constraint。
3. 驗目標值與關鍵變數的單位、量級。
4. LP bound sanity：max 整數解 ≤ LP bound；min 反之。

四步全過並對得上 Model.md 小例才可宣稱完成。

</verification>

<fatal_implications>

- NEVER 自行詮釋 Model.md；有歧義就停下退回 Phase 1。
- NEVER 常數位出現裸數字（含結構常數與迴圈邊界）。
- NEVER 用轉呼叫 helper／local function 包裝 `Program.cs` 的組裝順序。
- NEVER 手寫 `: VariableBase` / `: ParameterBase`，NEVER 自創資料夾或子 namespace。
- NEVER 在 `Dataload(IDataSource)` 生成、補值或判斷資料。
- NEVER 換 DLL 來源或改框架本體。
- NEVER fix loop 超過 5 次。

</fatal_implications>
