# OptimFoundation API — 端到端開發規範

> **這份文件是什麼**：用 OptimFoundation（封裝 IBM ILOG CPLEX 的 C# 套件）寫一個最佳化專案，從「建專案」到「寫模型」到「寫實驗」的完整標準流程。
> **給誰看**：第一次用這套套件的人，以及要用它產 code 的 AI。
> **怎麼用**：從 §1 一路往下做，每節結尾的 checklist 過了才進下一節。API 簽名有疑慮 → 查 `$OPT/AI-Modeling/CPLEX_API_REFERENCE.md`，**NEVER 憑記憶發明方法名**。
> **前置**：數學模型必須先在 `Model/<Project>_Model.md` 定完並經確認（三階段 phase gate 見 `CLAUDE.md`）。本檔只管 Phase 2（轉譯）與 Phase 3（實驗）。

`$OPT` = `C:/Users/zxcbi/Desktop/Projects/OptimizationFramework`。

### Canonical 邊界（AI MUST 先判斷）

- **本 guide 是專案結構與寫法的唯一權威**。結構、命名、namespace、注入方式、允許的 API 一律以本檔為準。
- **權威順序**：本 guide 的規則 → `AI-Modeling/CPLEX_API_REFERENCE.md` 的實際簽名 → 任何既有專案 code。既有專案與本檔衝突 → 視為待遷移，NEVER 反向修改本檔去迎合它。
- **NEVER 從既有專案推導結構**。`$OPT/AI-Modeling/Template/`、`$OPT/AI-Modeling/Projects/*`、`$OPT/OptimFoundation/OptimFoundation/Templates/*` 都**早於本版規範**，含已被禁止的寫法（手寫 `VariableBase`、`Sets.cs` 集中檔、子 namespace、`CreateXxx(rhs, name)`、`ProjectReference`）。它們可讀來理解 API 行為，NEVER 複製其結構。
- **只有一條 paved path**：generator（`[OptSet<T>]` / `[OptParam]` / `[OptVar]` + `[OptDim]`）。**手寫 `: VariableBase` / `: ParameterBase` 已廢止，沒有後路。**

---

## §0 心智模型

### 0.1 一條單向鏈

**資料的前提**：`Data/*.csv` 已經按框架標準形狀放在 input 位置。`Dataload` 的工作就只是把它們讀成積木——不生成、不補值、不判斷。

```text
① 資料層   Data/*.csv                     input，已就位
           Set_*（維度）+ Parameter_*（係數與結構常數）
           └ Data/Dataload.cs : DataContext
             └ OptData.Load(...)          唯一建構入口 → 驗證不過當場丟例外
② 變數層   VariableB_/X_/I_*              決策變數宣告（前綴決定型別）
             └ engine.BuildVars<T>        把宣告 × sets 展開成實際變數
③ 模型層   ObjectiveFunction              目標式（MUST 先建）
           Constraint_*                   限制式（AddLHS / AddRHS / CreateXxx）
④ 組裝     Program.cs 的 OptModel chain   唯一組裝點，唯一知道 Dataload 的地方
⑤ 執行環境 OptProject                     一個模型 × 一組設定；可掛 OnSolved
           OptExperiment                  m 個模型 × n 組 solver 設定
⑥ 解讀層   Solution/<Project>Solution.cs  ReadAndValidate + Print
```

資料來源若是不規則格式（9×9 矩陣、原始報表），用 `import` 模式先攤平成標準 CSV，見 §2.0。那是**選用的前置步驟**，不改變 ① 的前提。

### 0.2 三條不可協商的分工

**組裝只在 `Program.cs`。** 每種變數一行 `.AddVariables(...)`、目標式一行 `.AddObjective(...)`、每條限制式一行 `.AddConstraints(...)`。
Why: pipeline 本身就是模型組成清單，review 從這一個檔就能讀完模型由哪些部分組成；轉呼叫的 helper 或 local function 會把組裝順序藏起來。

**`Dataload` 只准出現在 `Program.cs`。** Objective / Constraint 的建構子逐項列出它實際用到的 Set、Parameter、scalar。
Why: 建構子簽名就是依賴清單，看簽名就知道這條式子碰哪些資料，測試也只需準備那幾樣。

**`OptEngine` 由 `Build(OptEngine engine)` 傳入，NEVER 進建構子。**
Why: 建構子只放資料依賴，engine 是執行期物件；混在一起就分不出「這個類別需要什麼資料」。

`OptData.Load` 完成後把資料視為唯讀。`DataContext.Freeze()` 只保護框架受控的 mutation API；直接修改 public field 或 mutable `List` 不保證立即攔截，因此專案 code MUST 不做這些寫入。

---

## §1 建立專案

### 1.1 專案結構（八個資料夾，NEVER 自創層級）

```text
Projects/<Project>/
├── <Project>.csproj
├── Program.cs        進入點：三態 CLI + 材料 + OptModel 組裝 + config + runner
├── Model/            <Project>_Model.md（含術語表的唯一模型文件）—— 不放 Glossary.md 或 .cs
├── Set/              Set_*.cs（一顆一檔）
├── Parameter/        Parameter_*.cs（一份一檔）
├── Variable/         Variable[B|X|I]_*.cs（一支一檔）
├── Objective/        ObjectiveFunction.cs
├── Constraint/       Constraint_*.cs（一條一檔）
├── Solution/         <Project>Solution.cs
└── Data/             Dataload.cs + Set_*.csv + Parameter_*.csv + raw/（不規則原始資料，選用）
```

- **NEVER 增減資料夾**。沒有 `Common/`、`Helpers/`、`Utils/`、`Services/`、`Models/`。
- **`Model/` 只放文件**（`.md`），NEVER 放 `.cs`；模型組裝碼在 `Program.cs`。
- **`Dataload.cs` 在 `Data/`**，與它讀的 CSV 同一層。
- **一個型別一個 `.cs` 檔，檔名 = 類別名** —— NEVER 把多顆 Set 塞進 `Sets.cs`、NEVER 把多條限制式塞進一個檔 —— Why: 一檔一型別才能靠檔名直接定位（`Set_Date` 就在 `Set/Set_Date.cs`）；集中檔改一顆就動到全部人的 diff。

### 1.2 Namespace 與型別修飾（全專案統一）

```csharp
// Set/Set_Item.cs
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>可生產的品項；對應 Model.md 的 ITEM。</summary>
    [OptSet<string>]
    public sealed partial class Set_Item { }
}
```

| 規則 | 說明 |
| --- | --- |
| namespace | **全專案一律 `<Project>`，沒有子 namespace** —— NEVER `MyProject.Set` / `MyProject.Data` —— Why: 子 namespace 只會讓每個檔多一排 `using`，分類已由資料夾表達 |
| namespace 寫法 | **block namespace `namespace X { }`** —— NEVER file-scoped `namespace X;` |
| 型別修飾 | **一律 `sealed`**。generator 型別寫成 `sealed partial`（`sealed` 只需寫在手寫那半，generator 產的那半不必重複） |
| `<summary>` | **每個型別 MUST 有**，一句話寫它對應 Model.md 的哪個符號 |
| attribute | **逐行獨立寫** —— NEVER 與 class 同一行 —— Why: 加第二個 attribute 時不必重排，宣告與修飾一眼分得開 |

### 1.3 命名總表（Model.md 符號 → code）

| 元素 | 類別名（PascalCase） | `Dataload` 欄位名（小寫元件開頭） | 檔案 |
| --- | --- | --- | --- |
| Set | `Set_Item` | `set_Item` | `Set/Set_Item.cs` |
| Parameter | `Parameter_Demand` | `parameter_Demand` | `Parameter/Parameter_Demand.cs` |
| Binary 變數 | `VariableB_Assign` | — | `Variable/VariableB_Assign.cs` |
| Continuous 變數 | `VariableX_Shortage` | — | `Variable/VariableX_Shortage.cs` |
| Integer 變數 | `VariableI_Batch` | — | `Variable/VariableI_Batch.cs` |
| 限制式 | `Constraint_Coverage` | — | `Constraint/Constraint_Coverage.cs` |

- NEVER 用 `ITEM` / `ROW` 全大寫欄位名，NEVER 用 `SetA` 這種 PascalCase 欄位名 —— ALWAYS `set_<語意>` / `parameter_<語意>` —— Why: `data.set_Item` 一眼看得出是哪種積木，且與類別名區分得開
- Set 成員字串 PascalCase 單數：`"Truck"` ✅、`"truck"` ❌、`"Trucks"` ❌

### 1.4 csproj（照抄，只改三處名字）

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>MyProject</RootNamespace>
    <AssemblyName>MyProject</AssemblyName>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
  </PropertyGroup>

  <ItemGroup>
    <Compile Remove="Generated/**/*.cs" />
  </ItemGroup>

  <ItemGroup>
    <None Update="Data\**\*.csv" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <Analyzer Include="..\..\dlls\OptimFoundation.Generators.dll" />
  </ItemGroup>

  <ItemGroup>
    <Reference Include="ILOG.Concert"><HintPath>..\..\dlls\ILOG.Concert.dll</HintPath></Reference>
    <Reference Include="ILOG.CPLEX"><HintPath>..\..\dlls\ILOG.CPLEX.dll</HintPath></Reference>
    <Reference Include="NLog"><HintPath>..\..\dlls\NLog.dll</HintPath></Reference>
    <Reference Include="OptimFoundation.Core"><HintPath>..\..\dlls\OptimFoundation.Core.dll</HintPath></Reference>
    <Reference Include="OptimFoundation.Cplex"><HintPath>..\..\dlls\OptimFoundation.Cplex.dll</HintPath></Reference>
  </ItemGroup>

</Project>
```

- 三處要改：`RootNamespace`、`AssemblyName`、資料夾名與 `.csproj` 檔名 → `<Project>`
- `Generated/` 是 generator 產碼落地供人檢視，MUST `Compile Remove`，否則與編譯期輸出撞名
- `Data\**\*.csv` MUST 複製到輸出，否則執行時在 bin 讀不到 input
- DLL 唯一來源是 `$OPT/AI-Modeling/dlls/` —— NEVER 用 NuGet、NEVER 用 `ProjectReference` 指向框架 src、NEVER 從 CPLEX Studio 安裝路徑抓 DLL —— Why: `ProjectReference` 只在框架 repo 內部成立，`Projects/` 底下編不過；指向安裝路徑則綁死本機環境。**限定例外**：`$OPT/OptimFoundation/OptimFoundation/Templates/*` 位於框架 repo 與 solution 內，MUST 保留 framework source / generator 的 `ProjectReference` 及 solver 的 `$(CplexDir)` HintPath，讓 template 隨框架 API 一起 build；例外不得複製到 `$OPT/AI-Modeling/Projects/*`。

**✅ 過關條件**：`dotnet build` 成功（此時還沒寫模型，只驗環境）。

---

## §2 資料層 — Set / Parameter / Dataload

轉譯順序固定：**Set → Parameter → Dataload**。

**Dataload 的唯一目標：把已就位的 CSV 讀成可驗證、可直接注入模型的積木。** 它是資料來源與模型之間的 adapter，不是資料生成器、不是商業邏輯層、不是第二個建模層。

### 2.0 input CSV 的形狀

`Data/*.csv` 是模型輸入的事實來源，形狀由 Model.md 的維度決定：

| 模型元素 | CSV | 範例 |
| --- | --- | --- |
| `Set_Item` | `Data/Set_Item.csv`；一列一個成員、無表頭 | `ItemA`、`ItemB` |
| `Parameter_Demand{Item,Date}` | `Data/Parameter_Demand.csv`；表頭 `ITEM,DATE,QTY`，一列一個維度組合 | `ItemA,2026-01-01,10` |
| scalar `Parameter_ShortagePenalty` | `Data/Parameter_ShortagePenalty.csv`；只有 `QTY` 表頭與**恰好一筆**值 | `QTY` / `1.0` |
| `Parameter_PreAssign{Item}`（`HasValue=false`） | 表頭只放 key 維度，不放 `QTY` | `ItemA` |

- MUST 先用維度思考：CSV 每個 key 欄對應一個 `[OptDim<Set_X>("Name")]`；數值欄才是 `QTY`
- MUST 讓 CSV 成為模型輸入的可檢視事實來源；換一批資料只換 CSV，**Dataload、Objective、Constraint 一行都不改**

**不規則來源 → `import` 模式（選用）**：來源是 9×9 矩陣、原始報表這類非標準形狀時，寫一個 `Dataload(string rawFile)` ctor 把它攤平，跑一次 import 產出上表的標準 CSV：

```powershell
dotnet run -- import raw/Puzzle_SHC279
```

原始檔放 `Data/raw/`，產出的 CSV 落在 `bin/.../Data/`（`FolderDir.Data`）。**這是資料準備工序，不是求解流程的一部分**——CSV 一旦就位，求解模式就只讀不算。

### 2.1 Set 積木 — 一顆一個檔

```csharp
// Set/Set_Item.cs
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>可生產的品項；對應 Model.md 的 ITEM。</summary>
    [OptSet<string>]
    public sealed partial class Set_Item { }
}
```

```csharp
// Set/Set_Date.cs
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>規劃期間的每一天；對應 Model.md 的 DATE。</summary>
    [OptSet<DateTime>]
    public sealed partial class Set_Date { }
}
```

- 合法元素型別：`string` / `DateTime` / `int` / `long` / `double` / `decimal`
- NEVER 寫裸 `[OptSet]` —— ALWAYS 顯式 `[OptSet<string>]` —— Why: 兩者產碼相同，但顯式讓元素型別在宣告處一眼可見
- NEVER 用裸 `List<string>` 當 Set —— Why: 積木化才會被 `DataContext` 註冊進驗證，裸 List 永遠驗不到

`SetBase<T>` 本身是 `IReadOnlyList<T>`，可直接 `foreach` / LINQ / 丟進 `BuildVars`，不必另存 `List` 視圖。四道防呆會丟例外：未載入即用、載入後為空、二次載入、成員重複。

### 2.2 Parameter — 一份一個檔

```csharp
// Parameter/Parameter_Demand.cs — 含值參數：generator 補 Item / Date / QTY
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>各品項各日的需求量；對應 Model.md 的 DemandQty_{Item,Date}。</summary>
    [OptParam]
    [OptDim<Set_Item>("Item")]
    [OptDim<Set_Date>("Date")]
    public sealed partial class Parameter_Demand { }
}
```

```csharp
// Parameter/Parameter_PreAssign.cs — 純 key 參數（沒有數值，只表示「這組合存在」）
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>已預先指派的品項；對應 Model.md 的 PREASSIGN。</summary>
    [OptParam(HasValue = false)]
    [OptDim<Set_Item>("Item")]
    public sealed partial class Parameter_PreAssign { }
}
```

```csharp
// Parameter/Parameter_ShortagePenalty.cs — 零維 scalar：generator 只補 QTY
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>短缺一單位的懲罰成本；對應 Model.md 的 ShortagePenalty。</summary>
    [OptParam]
    public sealed partial class Parameter_ShortagePenalty { }
}
```

| 規則 | 說明 |
| --- | --- |
| 數值欄位名 | **一律 `QTY`**，NEVER `Quantity` / `Amount`（generator 生成的就叫 QTY） |
| property 名 | `[OptDim<TSet>("Name")]` 的字串，PascalCase，NEVER `ALL_CAPS` |
| attribute 順序 | = key 組成順序，改順序 = 改 key |
| 建構物件 | **只允許 object initializer**：`new Parameter_Demand { Item = item, Date = date, QTY = 5 }` |
| 位置式 ctor | generator 有產，但 **NEVER 使用** —— Why: 改 `[OptDim]` 順序時 `new Parameter_Demand(a, b)` 不會報錯，只會靜默把維度接錯 |
| 自寫 ctor | **NEVER** —— 框架用 reflection 讀屬性順序組 key |
| `[FullGrid]` | opt-in，只標「語意上必須全格覆蓋」的參數；MILP 資料多半稀疏，預設 NEVER 加 |

零維 scalar Parameter 沒有 `[OptDim]`，CSV 只有 `QTY` 欄且 MUST 恰好一筆；在 `Program.cs` 組裝時用 `.Single().QTY` 取值 —— NEVER 用 `FirstOrDefault` —— Why: 筆數錯誤要立即失敗，不是靜默吃掉資料問題。

同一顆 Set 要在同一個類別取兩個角色名（`LotFrom` / `LotTo`）→ 只有 `[OptDim]` 做得到。
逃生口 `[OptParam<Set_A, Set_B>]` 與字串式 `[OptParam("Item", "Date:DateTime")]` 雖可編譯，**NEVER 用於新 code**。

### 2.3 結構常數也是 Parameter（天條）

模型裡的維度／結構常數（Sudoku 的 3×3、時間窗長度 7、班別數）**一律做成 Set 或 Parameter** —— NEVER 寫成迴圈邊界的字面數字。

❌ Bad

```csharp
for (int blockRow = 0; blockRow < 3; blockRow++)
    for (int columnOffset = 1; columnOffset <= 3; columnOffset++)
        engine.AddLHS(1.0, new VariableB_CellDigit { Row = blockRow * 3 + columnOffset, Column = column, Digit = digit });
```

✅ Good

```csharp
// Set/Set_Block.cs + Parameter/Parameter_BlockCell.cs（{Block,Row,Column}），從 CSV 讀進來
foreach (int block in blocks)
{
    foreach (var cell in blockCells.Where(c => c.Block == block))
        engine.AddLHS(1.0, new VariableB_CellDigit { Row = cell.Row, Column = cell.Column, Digit = digit });
}
```

Why: 寫死 3 的模型換成 4×4 或 16×16 就得改 code，違反「換資料不改 code」；做成資料後換 CSV 即可。迴圈邊界一律用 `foreach (var x in set)` 或 `set.Count`。

### 2.4 Dataload — 讀已就位的 CSV

```csharp
// Data/Dataload.cs
using OptimFoundation.Core;
using OptimFoundation.Core.IO;

namespace MyProject
{
    /// <summary>資料唯一入口：把 Data/*.csv 讀成積木，再由 DataContext 驗證。</summary>
    public sealed partial class Dataload : DataContext
    {
        public Set_Item set_Item = new();
        public Set_Date set_Date = new();
        public List<Parameter_Demand> parameter_Demand = new();
        public List<Parameter_ShortagePenalty> parameter_ShortagePenalty = new();

        /// <summary>無參數 ctor 只做一件事：把預設來源餵給下面真正讀檔的 ctor。</summary>
        public Dataload() : this(new CsvDataSource()) { }

        /// <summary>標準接口：一行載一顆，只讀不算。換 CSV / Oracle / 記憶體只換 source。</summary>
        public Dataload(IDataSource source)
        {
            set_Item.Load(source, "Set_Item");
            set_Date.Load(source, "Set_Date");
            parameter_Demand = source.LoadParam<Parameter_Demand>("Parameter_Demand");
            parameter_ShortagePenalty = source.LoadParam<Parameter_ShortagePenalty>("Parameter_ShortagePenalty");
        }
    }
}
```

**`Dataload(IDataSource source)` 的敘述白名單 —— 只允許兩種句子，沒有第三種**：

1. `set_X.Load(source, "Set_X");`
2. `parameter_X = source.LoadParam<Parameter_X>("Parameter_X");`

- NEVER 在這個 ctor 出現 `for` / `foreach` / `Enumerable.Range` / `Random` / enum 掃描 / 日期運算 / `if` / 補值 / 預設值
  ALWAYS 把這些搬到 §2.0 的 import 階段，先產出 CSV
  Why: 輸入事實藏進 code 之後，換資料就得改 code，review 也看不到實際跑的是什麼數字
- MUST `public sealed partial class Dataload : DataContext`（`partial` + 繼承缺一不可）—— Why: 少任一個，generator 的註冊碼不會產生，這顆 Dataload 永遠不會被驗證
- MUST 建構走 `OptData.Load(() => new Dataload())` —— NEVER 把裸 `new Dataload()` 當終點 —— Why: 裸 new 仍可編譯，但完全跳過參照完整性 / 重複 key / 數值 sanity 驗證，壞資料直接進 solver 產出「看起來最佳」的錯答案
- MUST 把 penalty、capacity、bound、Big-M 輸入等模型數值都做成 Parameter CSV —— NEVER 寫成 Dataload 的 hardcode public field
- NEVER 手寫驗證邏輯（`ValidateSetsCoverParameters()` 這種）—— 機械檢查集中在框架 `DataContext`
- NEVER `try/catch` 吞 `DataValidationException` —— 它刻意一次列出全部問題後 fail-fast
- 由資料推導的比值（Big-M 之類）用 `Numeric.SafeRatio(分子, 分母, context: "BigM")`，NEVER 手寫裸除法

**Set 載入方式**：

| 方式 | 寫法 | 何時用 |
| --- | --- | --- |
| **`Load(source, name)`** | `set_Item.Load(source, "Set_Item");` | **唯一寫法**，名稱 MUST 顯式帶上 |
| `LoadFrom` | `set_Item.LoadFrom(...);` | 只允許兩處：§2.0 的 import ctor 內攤平 raw；DB query-only 來源（第一引數是 SQL 不是名稱） |
| `LoadInline` | — | **NEVER**（正式專案一律落成 CSV） |
| `LoadCsv` | — | **NEVER**（繞過 `IDataSource`，換不了來源） |

- MUST 顯式帶 set 名 —— NEVER 依賴省略時的預設推導 —— Why: 檔名與類別名脫鉤時（改名、複用）省略式會靜默讀錯檔
- NEVER 用 `LoadFrom(parameter_X.Select(...).Distinct())` 從參數反推 set —— Why: 反推出來的 set 只有「參數表裡出現過」的成員，題目允許但這批資料沒用到的維度會靜默消失，變數少建、約束少建，解出來看起來正常但根本不是原題
- 元素型別非 string（`DateTime` / `int` / `double`…）不必自己 parse，`Load` 會依 `T` 自動轉型

**import ctor 與 `Export()`（只有不規則來源才需要）**

```csharp
/// <summary>import 模式：攤平 Data/raw/ 的不規則來源。rawFile 相對於 Data/、不帶副檔名。</summary>
public Dataload(string rawFile)
{
    var grid = CsvCtrl.ReadMatrixCsv(rawFile);
    // 依題目形狀攤平成積木；這是全專案唯一允許出現迴圈與 LoadFrom 的地方
}

/// <summary>把積木寫成標準 CSV，成為求解模式的 input。檔名 MUST 與上面讀取時一致。</summary>
public void Export()
{
    CsvCtrl.WriteSet(set_Item, "Set_Item");
    CsvCtrl.WriteParam(parameter_Demand, "Parameter_Demand");
}
```

**✅ 過關條件**：

- `OptData.Load(() => new Dataload())` 不丟例外，且各 Set 的 `Count` 與題目一致
- Model.md 的每顆 Set / Parameter 都能對到一個 CSV 與一行載入敘述
- `Dataload(IDataSource)` 內只有白名單那兩種句子
- 換一批合法 CSV 後，不必修改任何 `.cs`

---

## §3 變數層 — 一支一個檔

```csharp
// Variable/VariableB_Assign.cs
using OptimFoundation.Modeling;

namespace MyProject
{
    /// <summary>品項是否在該日生產；對應 Model.md 的 Assign_{Item,Date} ∈ {0,1}。</summary>
    [OptVar]
    [OptDim<Set_Item>("Item")]
    [OptDim<Set_Date>("Date")]
    public sealed partial class VariableB_Assign { }
}
```

**前綴是 load-bearing，不是命名慣例**：

| 前綴 | 型別 | `BuildVars<T>` 給的界限 |
| --- | --- | --- |
| `VariableB_` | Binary | `[0, 1]` |
| `VariableX_` | Continuous | `[0, 1E100]`（1E100 = 框架的「無上限」慣用值，不是 solver 的 infinity 常數） |
| `VariableI_` | Integer | `[0, 1E100]` |

- **建立變數只允許 `engine.BuildVars<T>(sets...)`** —— NEVER 用 `BuildBVs` / `BuildCVs` / `BuildIVs` —— Why: 型別過去在前綴與 Build 方法名兩處各表述一次可互相矛盾，收斂成前綴單一真相後才驗得了
- **界限一律寫成 constraint**。`BuildVars<T>` 沒有 bounds overload，預設就是 lb = 0、上界無限。Model.md 寫 `Produce_Item ≤ Capacity_Item` 就老實建一條 `Constraint_Capacity` —— Why: 界限本來就是 Model.md 的一條式子，藏進 `BuildCVs(0, 100, ...)` 的參數就無法逐條對回去驗證
- **NEVER 手寫 `: VariableBase` 並自己宣告 property** —— ALWAYS 用 `[OptVar]` + `[OptDim]` —— Why: 手寫的 property 順序與 `[OptDim]` 沒有單一真相，接錯不報錯
- 前綴非 `B_`/`X_`/`I_` → compile error `OPTF001`（錯誤訊息會教正確取名）
- NEVER 在 `[OptVar]` 參數指定型別（該參數已移除）
- 0 維純量變數：只掛光桿 `[OptVar]`，不加任何 `[OptDim]`
- 變數名格式：`ClassName@dim1@dim2`，分隔符是 `@`，**NEVER `|`**

**MUST 傳入 sets 的順序 = `[OptDim]` 宣告順序** —— Why: 順序錯了不會報錯，只會靜默把索引接錯，最後解出一個「別的題目」的答案。

---

## §4 模型層 — Objective 與 Constraint

### 4.1 Pool API（整個套件的核心手感）

`AddLHS` / `AddRHS` 是「往池子裡丟」，可以反覆呼叫；`CreateXxx` 是「出池」，出完自動清空、接著寫下一條。

```csharp
engine.AddLHS(coef, new VariableX_Produce { Item = item }); // LHS 變數項
engine.AddLHS(constant); // LHS 常數項
engine.AddRHS(coef, new VariableB_Open { Item = item }); // RHS 變數項（框架自動移項）
engine.AddRHS(constant); // RHS 常數

engine.CreateLessEqual($"{ConstraintName}@{item}"); // <=
engine.CreateGreatEqual($"{ConstraintName}@{item}"); // >=
engine.CreateEqual($"{ConstraintName}@{item}"); // ==
engine.CreateRange(lb, ub, $"{ConstraintName}@{item}"); // lb <= LHS <= ub（只吃 LHS）
```

**只允許單參數的 `CreateXxx(name)`。`CreateEqual(rhs, name)` / `CreateLessEqual(rhs, name)` / `CreateGreatEqual(rhs, name)` 這三個 overload 存在但 NEVER 使用** —— ALWAYS 用 `AddRHS(常數)` 表達右側 —— Why: 那三個 overload 是**覆蓋** `_rhsConst` 而不是累加。先 `AddRHS(demand)` 再 `CreateEqual(1.0, name)`，先前加的 demand 會被靜默丟掉，模型變成另一題。

**天條：Model.md 左邊的項進 `AddLHS`、右邊的項進 `AddRHS`，`>=` 就用 `CreateGreatEqual`。NEVER 自行移項 / 改號 / 翻轉方向 / 合併化簡** —— Why: 轉譯必須能逐條對回 Model.md 驗證，動過手腳就驗不了。

限制式群組與實際建立數量由 `EngineBase` 自動記錄（`engine.Solve()` 求解前會輸出完整 build summary）。`ConstraintBase.ConstraintCount` 已 obsolete —— NEVER 手動 `ConstraintCount++`、NEVER 自己印計數 —— Why: 第二份計數只會與框架的那份對不上。

### 4.2 一條限制式 = 一個檔

```csharp
// Constraint/Constraint_MaxDays.cs
using OptimFoundation.Core;
using OptimFoundation.Cplex;

namespace MyProject
{
    /// <summary>∀ item ∈ ITEM：Σ_date Assign[item][date] ≤ MaxDays[item]</summary>
    public sealed class Constraint_MaxDays : ConstraintBase
    {
        private readonly Set_Item items;
        private readonly Set_Date dates;
        private readonly List<Parameter_MaxDays> maxDaysByItem;

        public Constraint_MaxDays(
            Set_Item items,
            Set_Date dates,
            List<Parameter_MaxDays> maxDaysByItem)
        {
            this.items = items;
            this.dates = dates;
            this.maxDaysByItem = maxDaysByItem;
        }

        public void Build(OptEngine engine)
        {
            foreach (var item in items)
            {
                foreach (var date in dates)
                    engine.AddLHS(1.0, new VariableB_Assign { Item = item, Date = date });

                var maxDays = maxDaysByItem.FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0;
                engine.AddRHS(maxDays);

                engine.CreateLessEqual($"{ConstraintName}@{item}");
            }
        }
    }
}
```

- 繼承 `ConstraintBase` 取得 `ConstraintName`（= 類別名）
- class 開頭 `<summary>` MUST 貼上 Model.md 對應的數學式 —— Why: 驗收時逐條對照靠這行
- 限制式命名 MUST `ConstraintName@index1@index2`；`DateTime` 索引用 `{date:yyyy_MM_dd}`；複合索引用底線（`@{block}_{digit}`）
- Constructor MUST 逐項列出實際依賴，型別用**框架實際型別**（`Set_Item`、`List<Parameter_MaxDays>`、scalar `double`）
  NEVER 用 `IReadOnlyList<int>` 這種退化型別 —— Why: 型別即文件，`IReadOnlyList<int>` 看不出是哪顆 Set，傳錯了編譯器不會擋
  NEVER 接收 `Dataload`，也 NEVER 另造 `ModelContext` 之類的整包物件繞過這條規則
- **`OptEngine` 只從 `Build(OptEngine engine)` 進來，NEVER 進建構子**
- 依賴新增或刪除時，constructor 簽名與 `Program.cs` 的 call site MUST 同步改

### 4.3 目標式

```csharp
// Objective/ObjectiveFunction.cs
using OptimFoundation.Cplex;

namespace MyProject
{
    /// <summary>min Σ_item ShortagePenalty × Shortage[item]</summary>
    public sealed class ObjectiveFunction
    {
        private readonly Set_Item items;
        private readonly double shortagePenalty;

        public ObjectiveFunction(Set_Item items, double shortagePenalty)
        {
            this.items = items;
            this.shortagePenalty = shortagePenalty;
        }

        public void Build(OptEngine engine)
        {
            foreach (var item in items)
                engine.AddLHS(shortagePenalty, new VariableX_Shortage { Item = item });

            engine.CreateMinimize(); // 或 CreateMaximize()
        }
    }
}
```

**目標式 MUST 是 Model.md `OBJ` 段的逐項機械轉譯。**

- NEVER 自創目標式、NEVER 自行加減項、NEVER 改權重、NEVER 改 min/max 方向
- Model.md 沒有 OBJ 段（純可行性問題）→ **停止 Phase 2，退回 Model Design 補目標式** —— NEVER 自行發明零係數目標式、NEVER 省略 `.AddObjective(...)` —— Why: 天條規定 Coding 是純機械轉譯；Model.md 缺什麼就回去補什麼，不在 code 這一層代替使用者做建模決定
- Objective MUST 先於所有 constraints 建立（框架依 variables → objective → constraints 階段順序套用）

### 4.4 係數來源（天條）

**字面數值只能出現在 `AddLHS` / `AddRHS` 的「係數位」（第一參數，且第二參數是變數），NEVER 出現在「常數位」（單參數形式）。**

✅ Good

```csharp
engine.AddLHS(1.0, new VariableB_Assign { Item = item, Date = date }); // Σ x 的 identity 係數
var profit = profitByItem.FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0;
engine.AddLHS(profit, new VariableX_Produce { Item = item });
engine.AddRHS(capacityValue); // 值來自 Parameter
```

❌ Bad

```csharp
engine.AddRHS(40.0); // 常數位裸數字
engine.AddLHS(8.0, new VariableX_Produce { Item = item }); // 係數位，但 8.0 是模型係數不是 identity
engine.AddLHS(profitByItem.First(p => p.Item == item).QTY, new VariableX_Produce { }); // LINQ 內嵌
```

Why: 裸數字讓「改一個係數」變成全專案搜尋；LINQ 內嵌則是中間值 debug 不出來、`First` 沒資料直接炸。
**即使題目只提到兩三個數字，也 MUST 先定義成 Parameter 再經 Dataload 取用。**

### 4.5 Phase 2 禁止 soft constraint

正式模型的 `Constraint_*.cs` MUST 只使用 hard constraint API：`CreateLessEqual` / `CreateGreatEqual` / `CreateEqual` / `CreateRange`。

- NEVER 在 Phase 2 或 canonical 組裝 pipeline 使用 `CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft`
- NEVER 因為模型 infeasible、需求不確定或想讓 solver「比較容易有解」就自行改成 soft；這會改變數學模型，不是實作細節
- Soft constraint 只屬於 Phase 3 的**模型結構實驗**，必須由使用者明確要求，並建立具名的 experiment variant
- 實驗 soft variant 不得覆蓋、取代或混入 canonical hard model；報告結果時 MUST 標出被放鬆的限制式、penalty 與違反量

換句話說：**寫限制式時一律 hard；做實驗時才可能 soft。**

---

## §5 Program.cs — 唯一組裝點 + 三態 CLI

所有層級關係直接寫在這一個檔，平坦分成三段：**材料 → 模型 → 環境**。每種變數、目標式與每條限制式各占一個 fluent call。

```csharp
using OptimFoundation.Core;
using OptimFoundation.Cplex;

namespace MyProject
{
    internal static class Program
    {
        private static int Main(string[] args)
        {
            // 模式 1：import —— 攤平 Data/raw/ 產出標準 CSV（只有不規則來源才需要）
            if (args.Length >= 2 && args[0] == "import")
            {
                OptData.Load(() => new Dataload(args[1])).Export();
                return 0;
            }

            // exp 的 log 檔名 MUST 在第一次寫入前設定，整次執行才收在同一包
            bool isExperiment = args.Any(arg => string.Equals(arg, "exp", StringComparison.OrdinalIgnoreCase));
            if (isExperiment)
                Logging.SetLogFileName("MyProject_exp");

            // ── 1. 材料 ────────────────────────────────────────────
            var data = OptData.Load(() => new Dataload());
            double shortagePenalty = data.parameter_ShortagePenalty.Single().QTY;

            var projectConfig = new ProjectConfig
            {
                ProjectName = "MyProject",
                EnableSolverLog = true,
                ExportLP = true,
                ExportSol = true,
                DataId = "MyProject",
                UserId = "SYSTEM",
            };
            // 唯一 production baseline/champion；experiment clone 它，prod 直接使用它。
            var productionBaseline = new CplexConfig
            {
                epGap = 0.03,
                timeLimit = 300,
                workThreads = 8,
            };

            // ── 2. 模型 ────────────────────────────────────────────
            var model = new OptModel("Canonical")
                .AddVariables(engine => engine.BuildVars<VariableB_Assign>(data.set_Item, data.set_Date))
                .AddVariables(engine => engine.BuildVars<VariableX_Shortage>(data.set_Item))
                .AddObjective(engine => new ObjectiveFunction(data.set_Item, shortagePenalty).Build(engine))
                .AddConstraints(engine => new Constraint_MaxDays(data.set_Item, data.set_Date, data.parameter_MaxDays).Build(engine))
                .AddConstraints(engine => new Constraint_Coverage(data.set_Item, data.set_Date, data.parameter_Demand).Build(engine));

            // ── 3. 環境 ────────────────────────────────────────────
            // 模式 2：exp —— 掃 solver 設定，不做正式求解
            if (isExperiment)
            {
                var baseline = productionBaseline.Clone();
                var emphasis = baseline.Clone();
                emphasis.Emphasis = 2;

                var result = new OptExperiment("MyProject-tuning-r1", "baseline vs emphasis")
                    .AddModel(model)
                    .AddConfig("r1-baseline", baseline)
                    .AddConfig("r1-emphasis=optimal", emphasis)
                    .Run();

                foreach (var trial in result.Trials)
                    Logging.Info($"[Experiment] {trial.Label} status={trial.Metrics.Status} runTimeMs={trial.Metrics.RunTimeMs:F0}");
                return 0;
            }

            // 模式 3（預設）：正式求解
            using var project = new OptProject(model)
                .UseConfig(() => projectConfig)
                .UseConfig(() => productionBaseline)
                .OnSolved(engine => MyProjectSolution.ReadAndValidate(engine, data).Print());

            bool solved = project.Execute();
            return solved ? 0 : 1;
        }
    }
}
```

### 5.1 CLI 命令、`args` 與優先順序

`dotnet run` 後面的 `--` 是分隔符：`--` 前面屬於 `dotnet`，後面才會原樣傳入 `Main(string[] args)`；`--` 本身不會出現在 `args`。

三種正式支援的命令如下，一次只選一種：

| 模式 | 命令 | `args` | 結果 |
| --- | --- | --- | --- |
| production | `dotnet run --project <project.csproj>` | `[]` | 載入 canonical CSV，正式求解並執行 `OnSolved` |
| experiment | `dotnet run --project <project.csproj> -- exp` | `["exp"]` | 建立同一份 canonical model，執行 `OptExperiment`，不再執行 production runner |
| import | `dotnet run --project <project.csproj> -- import <raw>` | `["import", "<raw>"]` | 讀不規則來源、`Export()` canonical CSV，完成後立即 `return 0`，不建模也不求解 |

來源路徑含空白時 MUST 加引號，例如 `-- import "raw/My Puzzle.csv"`，它仍是單一 `args[1]`。

判斷順序是程式行為的一部分：

1. 先檢查 `args.Length >= 2 && args[0] == "import"`；命中後執行 import 並立即結束。
2. 未命中 import 才掃描所有參數是否含不分大小寫的 `exp`。
3. 前兩者都未命中才進 production。

因此非標準組合的實際結果如下；這些寫法不屬正式 CLI contract，NEVER 用它們串接工作流程：

| 寫法 | 實際結果 | 原因／正確替代 |
| --- | --- | --- |
| `-- import <raw> exp` | **只 import** | import 分支先命中且 `return 0`；若要接著實驗，分兩次執行 `-- import <raw>` 與 `-- exp` |
| `-- exp import <raw>` | **只 experiment** | `args[0]` 不是 `import`，但掃描到 `exp`；其餘參數不會成為 import 輸入 |
| `-- EXP` / `-- Exp` | experiment | `exp` 使用 `OrdinalIgnoreCase` 比較 |
| `-- Import <raw>` | production | `import` 判斷目前區分大小寫；正式寫法固定使用小寫 `import` |
| `-- import` | production | 少了 `args[1]`，不符合 import 條件；正式寫法 MUST 提供來源 |
| `-- <unknown>` | production | 未命中 import 或 exp；正式工作流程不要傳未定義參數 |

三態沒有「import 後自動 solve／experiment」的複合模式。需要連續操作時 MUST 分成兩次命令，讓第一步產出的 canonical CSV 成為第二步明確可檢視的輸入：

```powershell
dotnet run --project <project.csproj> -- import <raw>
dotnet run --project <project.csproj> -- exp
```

| 規則 | 說明 |
| --- | --- |
| 三態互斥 | `import <raw>` / `exp` / 無參數，一次只做一件事 |
| exit code | 求解成功 0、失敗 1；import 完成 0 |
| `Logging.SetLogFileName` | 只在 exp 模式呼叫，且 MUST 在 `OptData.Load` 之前 —— Why: `OptExperiment` 自己不換 log 檔名，晚設會留下只有載入摘要的殘檔。格式固定 `{name}_{時間戳}.txt`，時間戳只能在最後 |
| 每個組成各占一行 | 每種變數一行 `.AddVariables(...)`；Objective 一行 `.AddObjective(...)`；每條 Constraint 一行 `.AddConstraints(...)` |
| 順序固定 | variables → objective → constraints |
| 新增一條限制式 | `Constraint/` 加一檔 + pipeline 加一行，沒有第三個地方要改 |
| scalar 取值 | 材料段用 `.Single().QTY` 取出，再逐項傳入 |
| `OptModel` 名稱 | canonical 模型一律 `"Canonical"`；Phase 3 的 variant 另取名 |
| 兩層 config 具名宣告 | `ProjectConfig` 管專案身分與輸出、`CplexConfig` 只管 solver 旋鈕 —— NEVER 用 config factory helper 把內容藏起來 |
| `OptProject` | production runner，套兩層 config、執行一次；`OnSolved` 只掛在這裡 |
| `OptExperiment` | experiment runner，展開 model × config，自動儲存 |
| NEVER 在 fluent call 直接寫 `AddLHS` / `AddRHS` | pipeline 只列組成；數學式留在各 Objective / Constraint class |
| NEVER 把 `Dataload` 傳進 Objective / Constraint | pipeline 逐項傳它實際使用的 Set / Parameter / scalar |
| NEVER 包裝組裝流程 | 禁止只負責轉呼叫的類別／函式／local function；review 必須能從這一條 chain 讀完模型組成 |
| 不重印 framework log | Status / objective / IIS 路徑與 conflict names 由 `OptEngine` 自動記錄；專案端只印業務語意訊息 |

`productionBaseline` 是 tuning promotion 的唯一寫回點。AI 跑完實驗後若 champion 通過 §8 promotion gate，更新這顆 initializer 並重跑 production；NEVER 另建一份只給 prod 的 config，否則下一輪 experiment 會從過期 baseline 出發。

`ProjectConfig.EnableSolverLog` 預設 `true`，三個 export 預設 `false`；`DataId` / `UserId` 是 solution metadata 的預設值。`CplexConfig` 的 solver 欄位維持 camelCase；抽象旋鈕（`Emphasis` / `Seed` / `FeasibilityTol` 等）與相應欄位指向同一設定。

---

## §6 Solution — 取解、驗證、輸出

`Solution/<Project>Solution.cs` 是 MUST 的固定檔，掛在 `OnSolved`。**解驗證用一般 C# 程式做，NEVER 只憑 solver status 宣稱成功。**

解讀層是唯一允許接收整包 `Dataload` 的地方——`ValidateRules` 本來就要對照所有原始資料逐條驗證。

```csharp
// Solution/MyProjectSolution.cs
using OptimFoundation.Core;
using OptimFoundation.Core.IO;
using OptimFoundation.Cplex;

namespace MyProject
{
    /// <summary>讀回解、以題目規則驗證、列印並輸出 CSV。</summary>
    public sealed class MyProjectSolution
    {
        private readonly Dictionary<string, double> assign;

        private MyProjectSolution(Dictionary<string, double> assign) => this.assign = assign;

        public static MyProjectSolution ReadAndValidate(OptEngine engine, Dataload data)
        {
            Logging.Info($"Status={engine.Status} Obj={engine.GetObjectiveValue():F4} " +
                         $"BestBound={engine.BestObjValue:F4} MIPGap={engine.MIPGap:P2}");

            var assign = engine.GetSetVarValues<VariableB_Assign>();
            ValidateRules(assign, data);

            FolderDir.Solution.CreateFolder(); // MUST，否則 WriteSolution 丟 DirectoryNotFoundException
            CsvCtrl.WriteSolution<VariableB_Assign>(engine, "MyProject", "SYSTEM");
            return new MyProjectSolution(assign);
        }

        /// <summary>逐條把解代回 Model.md 的限制式；不成立就丟例外，NEVER 只記 log 繼續。</summary>
        private static void ValidateRules(Dictionary<string, double> assign, Dataload data)
        {
            foreach (var item in data.set_Item)
            {
                var maxDays = data.parameter_MaxDays.FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0;
                double used = data.set_Date.Sum(date =>
                    assign.TryGetValue($"VariableB_Assign@{item}@{date:yyyy-MM-dd}", out var value) ? value : 0.0);

                if (used > maxDays + 1e-6)
                    throw new InvalidOperationException($"{item} 違反 MaxDays：{used} > {maxDays}。");
            }
        }

        public void Print()
        {
            foreach (var kv in assign.Where(kv => kv.Value > 0.5))
                Console.WriteLine(kv.Key);
        }
    }
}
```

| 方法 | 回傳 |
| --- | --- |
| `GetObjectiveValue()` | `double` |
| `GetVariableValue(fullName)` | `double`；`fullName` 含 `@` |
| `GetSetVarValues<T>()` | `Dictionary<string, double>`，key = 完整變數名 |
| `GetSolution(typeName = null)` | `IReadOnlyDictionary`；`null` = 全部變數 |
| `GetBVSolution()` / `GetCVSolution()` / `GetIVSolution()` | 依型別分類的全部解 |
| `GetSetVarNames<T>()` / `GetAllVarNames()` | `string[]` |

輸出資料夾（都在 `bin/Debug/net8.0/` 底下）：`Data`（輸入）、`Solution`、`Logs`、`Models`（LP/MPS）、`Sols`、`IISs`、`Experiments`。
**`new ProjFolder(...)` 不會自動建資料夾**，任何寫檔前 MUST 先 `FolderDir.Xxx.CreateFolder()`。

---

## §7 驗收 — 解驗證協定

`dotnet run` 有輸出 ≠ 會動。MUST 依序四步，任一步不過就停下回報，NEVER 宣稱完成：

1. **Status 三分診斷**
   - `Optimal` → 進第 2 步
   - `Infeasible` → **MUST 去讀 `bin/Debug/net8.0/IISs/*.ilp`**（或 `GetConflictConstraints()`）拿最小衝突集；先自查 Big-M 是否太小、有無互斥硬約束 —— NEVER 靠猜、NEVER 改成 soft constraint 繞過
   - `Unbounded` → 某個方向漏了界；查該變數缺哪條上限 constraint（`BuildVars` 的上界是 1E100，界限一律靠 constraint 表達）
2. **可行性代回**：在 `Solution/<Project>Solution.cs` 的 `ValidateRules` 逐條把解值代回**每一條** constraint，確認 `LHS op RHS` 成立 —— Why: solver 回 Optimal 只保證「它解的那個模型」可行，不保證那個模型 = 你的題目
3. **單位一致性**：目標值與關鍵變數的單位跟題目一致（利潤 = 錢、產量 = 件、工時 = 時），量級合理
4. **LP bound sanity**：max 問題整數解 ≤ LP relaxation bound；min 問題 ≥。差太離譜 → 疑 Big-M 或係數錯

✅ Good：四步都過，且目標值對得上 Model.md 的手算小例
❌ Bad：看到 `Status == Optimal` 就回報「解出來了」

build 失敗走 fix loop：擷取 compiler error → 修 → 重 build，**至多 5 次**；仍失敗停下回報，NEVER 硬掰。

---

## §8 寫實驗 — 參數掃描

`OptExperiment` 將已定義的模型與具體 solver configs 展開成笛卡兒積，擷取 Trial，最後輸出 `Experiments/<name>.csv + .json`。同一份 `OptModel` 可先交給 `OptProject` 正式求解，再交給實驗 runner。

**同名實驗是 append，不是覆寫。** `Run()` 內的 `Save()` 會先讀既有 JSON，再把歷史 trials 合併回傳值的 `result.Trials`。因此 experiment 命名 MUST 用 `<Project>-tuning-r<N>`，每輪 N 加一 —— NEVER 重複用同一個名字 —— Why: 重跑同名 experiment 後直接 `foreach (result.Trials)` 會再次看到歷史資料，把舊 trial 誤報成本輪結果。

```csharp
var baseline = solverConfig.Clone();
var emphasis = baseline.Clone();
emphasis.Emphasis = 2;
var tighterGap = baseline.Clone();
tighterGap.epGap = 0.01;
var threads4 = baseline.Clone();
threads4.workThreads = 4;

var result = new OptExperiment("MyProject-tuning-r1", "一次只改一個 solver 旋鈕")
    .AddModel(model)
    .AddConfig("r1-baseline", baseline)
    .AddConfig("r1-emphasis=optimal", emphasis)
    .AddConfig("r1-gap=0.01", tighterGap)
    .AddConfig("r1-threads=4", threads4)
    .Run();
```

| 規則 | 說明 |
| --- | --- |
| 共用一份 data | 所有 cell 引用同一份 `OptData.Load` 結果，載入後視為唯讀 |
| experiment 預設安靜 | solver log、LP/MPS/Sol export 與 housekeeping 預設都 OFF |
| 具體 config | 用 `Clone()` 建 variant，NEVER 用 tune delegate 突變共用 baseline |
| 笛卡兒積 | `.AddModel` × `.AddConfig` 自動展開；單一 cell 用 `.AddTrial(model, label, config)` |
| label | 自動成為 `ModelName \| config-label`；輪次前綴（`r1-`）寫進 config label |
| 命名 | experiment name 一律 `<Project>-tuning-r<N>`，每輪遞增 |
| baseline 可重現 | 固定 `workThreads` / `parallelMode` / `randomSeed`，讓每次跑的停點一致 |
| `OnSolved` 邊界 | 只屬 `OptProject`；實驗不應大量寫 solution |
| 一次只動一個旋鈕 | 同時改兩個就分不出是哪個造成差異 |

### 8.1 AI 閉環：Trial → champion → production baseline

`OptExperiment` 本身只負責執行與保存證據，不會改傳入的 config，也不會替 production 選 winner。完成 tuning 的責任在 AI workflow：

```text
productionBaseline
  └─ Clone variants
       └─ OptExperiment.Run()
            └─ Trial.Config + Trial.Metrics
                 └─ AI promotion gate
                      └─ 更新 productionBaseline
                           └─ build + production + ValidateRules
```

每輪 MUST：

1. 使用新實驗名 `<Project>-tuning-r<N>`，只分析本輪 Trial；同名 append 的歷史結果不得混進本輪排名。
2. 先淘汰錯誤、Infeasible、Unbounded、無可行解或未達 objective / gap 品質門檻的 candidate。
3. 在相同 instance、seed 與資源限制下比較；正式 promotion 用 3–5 seeds 與 hold-out instances。第一個 solve 作 warm-up 並排除，variants 跨 seed 輪替或隨機化執行順序，避免固定讓 baseline 承擔 cold-start。runtime 用 shifted geometric mean，timeout 用 PAR10，改善幅度須大於 baseline variability。
4. 依序比較解的正確性與品質，再比較 runtime；node / iteration 只作診斷或 tie-break。NEVER 讓一個比較快但解較差的 trial 勝出。
5. 從 champion 的 `Trial.Config.SolverSpecific` 取得完整設定快照，由 AI 明確更新 `Program.cs` 的 `productionBaseline` initializer。同步更新 initializer 上方 provenance，至少寫來源 experiment + Trial label。NEVER 讓執行中的程式修改 source，NEVER 讓 prod 每次動態挑歷史上最快的一列。
6. 在專案根 `TuningHistory.md` 新增本輪紀錄：experiment、baseline / champion Trial、seeds / instances、彙總方法、Status / objective / gap、before/after config diff、決策與理由。該檔 MUST 納入 source control；`bin/Experiments/*.json` 會被清除，不能作唯一 provenance。
7. promotion 後 MUST `dotnet build` 並跑無參數 production；`OnSolved` / `ValidateRules`、Status、objective、gap 與輸出全數通過才完成升級，並把結果補回同一筆 history。失敗則保留原 baseline並記錄 rejected。
8. 成功 promotion 的 champion 成為下一輪 baseline；沒有可靠勝者也記錄 retain。連續 3 輪無實質改善就停止 solver tuning 或依 Phase 3 規範升級方向。

若沒有 candidate 可靠勝過 baseline，正確結果是保留既有 production baseline，而不是硬挑本輪牆鐘時間最小者。完成報告與 `TuningHistory.md` MUST 寫明 baseline、champion（若有）、before / after、seeds / instances、promotion 或不 promotion 的理由。

若需要覆寫 experiment 的 project-level defaults，可使用 `.UseConfig(() => projectConfig)`；factory 每個 cell 都會執行。一般 solver 掃描保留預設即可。

Soft constraint 不屬一般 solver tuning。若使用者明確要求，另建具名 `OptModel` variant 並保留 hard baseline；NEVER 覆蓋 canonical constraint。

**MUST 先過 §7 正確性 gate 才做 tuning。** 對錯的模型調參數只會更快得到錯答案。

---

## §9 API 速查卡

```csharp
// 材料（Program.cs §1 段）
var data = OptData.Load(() => new Dataload());
double penalty = data.parameter_ShortagePenalty.Single().QTY;

var projectConfig = new ProjectConfig
{
    ProjectName = "MyProject",
    EnableSolverLog = true,
    ExportLP = true,
};
var solverConfig = new CplexConfig { epGap = 0.0, timeLimit = 60, workThreads = 4 };

// 模型（Program.cs §2 段）
var model = new OptModel("Canonical")
    .AddVariables(engine => engine.BuildVars<VariableB_Assign>(data.set_Item, data.set_Date))
    .AddVariables(engine => engine.BuildVars<VariableX_Shortage>(data.set_Item))
    .AddObjective(engine => new ObjectiveFunction(data.set_Item, penalty).Build(engine))
    .AddConstraints(engine => new Constraint_Capacity(data.set_Item, data.parameter_Capacity).Build(engine));

// 環境（Program.cs §3 段）
using var project = new OptProject(model)
    .UseConfig(() => projectConfig)
    .UseConfig(() => solverConfig)
    .OnSolved(engine => MyProjectSolution.ReadAndValidate(engine, data).Print());
bool ok = project.Execute();

// Objective / Constraint 的 Build(engine) 內
engine.AddLHS(coef, new VariableX_Produce { Item = item });
engine.AddLHS(constant);
engine.AddRHS(coef, new VariableB_Open { Item = item });
engine.AddRHS(constant);
engine.CreateLessEqual($"{ConstraintName}@{item}");
engine.CreateGreatEqual($"{ConstraintName}@{item}");
engine.CreateEqual($"{ConstraintName}@{item}");
engine.CreateRange(lb, ub, $"{ConstraintName}@{item}");
engine.CreateLeSoft(rhs, penalty, name); // EXPERIMENT ONLY
engine.CreateGeSoft(rhs, penalty, name); // EXPERIMENT ONLY
engine.CreateEqSoft(rhs, penalty, name); // EXPERIMENT ONLY

// 目標式
engine.AddLHS(coef, new VariableX_Shortage { Item = item });
engine.CreateMinimize(); // 或 CreateMaximize()

// 取解（Solution/<Project>Solution.cs）
double obj = engine.GetObjectiveValue();
var dict = engine.GetSetVarValues<VariableB_Assign>();
var metrics = project.optEngine.LastMetrics; // gap / bound / node·iter / 軌跡

// 資料 I/O（Data/Dataload.cs）
set_Item.Load(source, "Set_Item");
parameter_Demand = source.LoadParam<Parameter_Demand>("Parameter_Demand");
CsvCtrl.WriteSet(set_Item, "Set_Item"); // 只在 import 模式
CsvCtrl.WriteParam(parameter_Demand, "Parameter_Demand"); // 只在 import 模式
FolderDir.Solution.CreateFolder();
CsvCtrl.WriteSolution<VariableB_Assign>(engine, "MyProject", "SYSTEM");

// 日誌：OptProject 會自動設定檔名；framework lifecycle 不在專案端重複印
Logging.Info("業務語意或解驗證訊息");
```

### 黑名單 — 這些寫了必爛，或已被本規範禁止

| ❌ 禁用 | ✅ 正確替代 | 原因 |
| --- | --- | --- |
| `engine.GetVarSol(name)` | `engine.GetVariableValue(name)` | 不存在 |
| `engine.GetSetVarSol<T>()` | `engine.GetSetVarValues<T>()` | 不存在 |
| `CSVCtrl.SaveToCSV<T>(...)` | `CsvCtrl.WriteSolution<T>(engine, dataId, userId)` | 不存在 |
| `CSVCtrl.xxx`（大寫 V） | `CsvCtrl.xxx` | 大小寫錯 |
| `FolderDir.Result` | `FolderDir.Solution` | 不存在 |
| `new OptEngineConfig { ... }` | `new CplexConfig { ... }` | 不存在 |
| `engine.AddPool` / `AddPoolRHS` | `engine.AddLHS` / `AddRHS` | 不存在 |
| `BuildBVs(typeof(T), sets)` | `BuildVars<T>(sets)` | 不存在 |
| `BuildBVs<T>` / `BuildCVs<T>` / `BuildIVs<T>` | `BuildVars<T>(sets)` | 型別應由前綴單一決定 |
| `BuildCVs<T>(lb, ub, sets)` 設界限 | 寫成一條 `Constraint_*` | 界限藏進參數就無法逐條驗證 |
| `CreateEqual(rhs, name)` 等三個 overload | `AddRHS(rhs)` + `CreateEqual(name)` | 該 overload **覆蓋**右側常數，混用會靜默丟資料 |
| `ConstraintCount++` / 自印計數 | 不寫，`EngineBase` 自動統計 | 已 obsolete，第二份計數會對不上 |
| 手寫 `: VariableBase` / `: ParameterBase` | `[OptVar]` / `[OptParam]` + `[OptDim]` | 已廢止，property 順序無單一真相 |
| `new Parameter_X(a, b)` 位置式 | `new Parameter_X { A = a, B = b }` | 改 `[OptDim]` 順序會靜默接錯 |
| `set.LoadCsv("...")` | `set.Load(source, "Set_X")` | 繞過 `IDataSource`，換不了來源 |
| `set.LoadInline(...)` | 落成 CSV | 資料藏進 code |
| `set.Load(source)` 省略名 | `set.Load(source, "Set_X")` | 改名時靜默讀錯檔 |
| `namespace MyProject.Data` 子 namespace | `namespace MyProject` | 全專案單一 namespace |
| file-scoped `namespace X;` | block `namespace X { }` | |
| `Model/` 放 `.cs` | 只放 `.md`；組裝碼在 `Program.cs` | |
| 假設變數名分隔符是 `\|` | 是 `@` | |
| 假設 `ProjFolder` 建構子會建資料夾 | MUST 手動 `FolderDir.Solution.CreateFolder()` | |
| `override Build()` / `override Solve()` | 已非 virtual；改覆寫 `BuildCore()` / `SolveCore()` | |
| `ProjectReference` 指向框架 src | `..\..\dlls\` HintPath | 只允許 `$OPT/OptimFoundation/OptimFoundation/Templates/*` 內部 template；AI-Modeling 專案一律禁用 |

---

## §10 常見錯誤與反模式

| 症狀 | 真正原因 | 修法 |
| --- | --- | --- |
| `CS0246` 找不到型別 | csproj `HintPath` 層數錯 | `Projects/` 底下一律 `..\..\dlls\` |
| generator 沒產碼 | 類別漏了 `partial`，或 `Set_*` 漏掛 `[OptSet<T>]` | 補上；漏掛且被 Dataload 引用會報 `OPTF006` |
| `OPTF001` | 變數前綴不是 `VariableB_/X_/I_` | 依前綴表改名 |
| 找不到 CSV | csproj 漏 `Data\**\*.csv` 的 copy 設定，或 CSV 根本沒放進 `Data/` | 補 copy 設定；不規則來源先跑 `dotnet run -- import raw/<file>` |
| `DirectoryNotFoundException` | 寫檔前沒 `CreateFolder()` | `FolderDir.Solution.CreateFolder()` |
| 解出來但數值離譜 | 變數 sets 傳入順序與 `[OptDim]` 不一致 | 逐一對齊順序 |
| 限制式右側值莫名消失 | 用了 `CreateEqual(rhs, name)` 覆蓋掉先前 `AddRHS` | 改回 `AddRHS(...)` + `CreateEqual(name)` |
| `Unbounded` | 界限沒寫成 constraint（`BuildVars` 上界是 1E100） | 補該變數的上限 `Constraint_*` |
| Infeasible 但模型看起來對 | Big-M 太小、或兩條硬約束互斥 | 讀 `IISs/*.ilp`；以 `ProjectConfig.ExportLP = true` 開 `.lp` 檔用肉眼查 |
| 改一個係數要改好幾個檔 | 係數 hardcode 在 Constraint 裡 | 全部移進 Parameter 的 `QTY` |
| 換資料還要改 `.cs` | `Dataload(IDataSource)` 裡有迴圈／判斷／補值 | 搬到 import ctor，先產 CSV |
| 重跑實驗看到舊資料 | 同名 experiment append | 改用 `<Project>-tuning-r<N>` 遞增命名 |

### 反模式

❌ `try { ... } catch (Exception) { throw; }` —— 什麼都沒做卻讓每個方法多兩層縮排，直接刪掉
❌ 在 Variable / Parameter 類別寫建構子 —— 框架用 reflection 組 key，自己寫只會干擾
❌ Objective / Constraint 建構子接收 `Dataload` —— 改成只注入該式子使用的 Set / Parameter / scalar（`Solution` 不在此限）
❌ 把 `OptEngine` 放進建構子 —— engine 只從 `Build(OptEngine engine)` 進來
❌ 在 Constraint class 手動 `ConstraintCount++` 或自印計數 —— `EngineBase` 已自動統計
❌ 用純轉呼叫 helper／local function 包裝模型組裝 —— 每個組成必須直接列在 `Program.cs` 的 chain
❌ 用 config factory helper 隱藏設定內容 —— 在 `Program.cs` 獨立建立具名 `ProjectConfig` 與 `CplexConfig`
❌ 在正式 `Constraint_*.cs` 使用 `CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft` —— soft constraint 僅供使用者明確要求的 Phase 3 結構實驗
❌ 在 `Dataload(IDataSource)` 用迴圈、`Random`、enum 或日期運算製造模型輸入 —— 移到 import 階段先產 CSV
❌ 迴圈邊界寫字面數字（`for (int i = 0; i < 3; i++)`）—— 結構常數做成 Set / Parameter，用 `foreach` 或 `set.Count`
❌ 改 OptimFoundation 框架本體（`dlls/` 是唯讀的）—— 缺方法就在專案端寫 helper / extension

---

## 相關文件

| 文件 | 用途 |
| --- | --- |
| `CLAUDE.md`（本資料夾） | 三階段 phase gate 天條 |
| `Model Design/CLAUDE.md` | Phase 1：怎麼把題目變成 Model.md |
| `Model Design/linearization-patterns.md` | 非線性 → 線性的 8 類手法 |
| `Foundation Coding/CLAUDE.md` | Phase 2 規範（本檔是它的擴充版教學） |
| `Foundation Tuning/CLAUDE.md` | Phase 3：solver / IIS / 模型結構三方向調校 |
| `$OPT/AI-Modeling/CPLEX_API_REFERENCE.md` | **API 簽名唯一權威**，有疑慮一律查這份 |
| `$OPT/AI-Modeling/tuning/CLAUDE.md` | CplexConfig 全旋鈕對照表 |
