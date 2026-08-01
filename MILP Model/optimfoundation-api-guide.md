# OptimFoundation API — 端到端開發規範

> **這份文件是什麼**：用 OptimFoundation（封裝 IBM ILOG CPLEX 的 C# 套件）寫一個最佳化專案，從「建專案」到「寫模型」到「寫實驗」的完整標準流程。
> **給誰看**：第一次用這套套件的人，以及要用它產 code 的 AI。
> **怎麼用**：從 §1 一路往下做，每節結尾的 checklist 過了才進下一節。API 簽名有疑慮 → 查 `$OPT/AI-Modeling/CPLEX_API_REFERENCE.md`，**NEVER 憑記憶發明方法名**。
> **前置**：數學模型必須先在 `Model/<Project>_Model.md` 定完並經確認（三階段 phase gate 見 `CLAUDE.md`）。本檔只管 Phase 2（轉譯）與 Phase 3（實驗）。

`$OPT` = `C:/Users/zxcbi/Desktop/Projects/OptimizationFramework`；起始範本在 `$OPT/AI-Modeling/Template`。

---

## §0 心智模型：一條直線，五個層級

整條鏈是**單向**的，沒有回頭邊。看懂這張圖就看懂整個套件：

```text
① 資料層   Set_*（維度）+ Parameter_*（係數）
           └ Dataload : DataContext        載入資料、輸出解
             └ OptData.Load(...)           唯一建構入口 → 驗證不過當場丟例外
                        ↓ dataload
② 變數層   VariableB_/X_/I_*               決策變數宣告（前綴決定型別）
             └ engine.BuildBVs/CVs/IVs     把宣告 × sets 展開成實際變數
                        ↓
③ 模型層   ObjectiveFunction               目標式（MUST 先建）
           Constraint_*                    限制式（AddLHS / AddRHS / CreateXxx）
                        ↓
④ 引擎層   CplexConfig → OptEngine         變數與限制式都住在這裡
                        ↓
⑤ 執行層   solve       OptModel.Execute()  一次求解
           experiment  Trial / Experiment  多組設定掃描
```

**②③ 的呼叫順序直接寫在 `Program.cs` 的 pipeline，不另外包 `VariableCreate` / `CreateVariables()` / `BuildModel` 類別或 local function。**
Why: 這些包裝只做轉呼叫、沒有模型邏輯，會把實際組裝順序藏起來。每種變數各用一行 `.AddVariables(...)`，Objective 與每條 Constraint 各用一行 `.AddModel(...)`，從 pipeline 就能完整看出模型由哪些部分組成。

**Solver 設定也要在 `Program.cs` 另外建立成具名的 `CplexConfig`，再交給 `.UseConfig(...)`；NEVER 用 `NewConfig()` helper 或把整段設定藏在 pipeline 裡。**
Why: 模型組成與求解器設定是兩件事。設定獨立後，review 時可以直接確認 gap、time limit、threads 與輸出開關，不必追進 helper。

**Objective / Constraint 不得接收或保存整包 `Dataload`。** 它用到哪一顆 Set、哪一份 Parameter、哪一個 scalar 常數，就在 `Program.cs` 的 `.AddModel(...)` pipeline 中建構時逐項傳入；`OptEngine` 同樣明確傳入。
Why: 類別的資料依賴直接寫在建構子簽名上，避免它任意碰觸整個資料層，也讓測試能只準備該式子真正需要的資料。

---

## §1 建立專案

1. 複製 `$OPT/AI-Modeling/Template/` → `$OPT/AI-Modeling/Projects/<Project>/`，改資料夾名與 `.csproj` 檔名
2. **改 `csproj` 兩處相對路徑**（複製後都多一層 `..\`，不改一定編不過）：DLL `HintPath` 與 generator `Analyzer` 全部 `..\dlls\` → `..\..\dlls\`
3. 改 `csproj` 的 `RootNamespace` / `AssemblyName` → `<Project>`
4. 全專案 namespace `Template.*` → `<Project>.*`

### 專案結構（csproj + Program.cs + 六個資料夾，NEVER 自創層級）

```text
Projects/<Project>/
├── <Project>.csproj
├── Program.cs        唯一進入點：solve / experiment 雙模式 + 所有層級組裝
├── Model/            <Project>_Model.md（數學模型）+ Glossary.md
├── Set/              Set_*.cs（一顆積木一檔）+ Dataload.cs
├── Parameter/        Parameter_*.cs
├── Variable/         VariableB_/X_/I_*.cs
├── Objective/        ObjectiveFunction.cs
└── Constraint/       Constraint_*.cs
```

Namespace = `<Project>.<資料夾名>`；`Program.cs` 用 top-level statements，不進 namespace。
**一個型別一個 `.cs` 檔，檔名 = 類別名**（Set / Parameter / Variable / Constraint 全部適用），attribute 逐行獨立寫。

### csproj 必要區塊

```xml
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
  <Analyzer Include="..\..\dlls\OptimFoundation.Generators.dll" />
</ItemGroup>

<ItemGroup>
  <Reference Include="ILOG.Concert"><HintPath>..\..\dlls\ILOG.Concert.dll</HintPath></Reference>
  <Reference Include="ILOG.CPLEX"><HintPath>..\..\dlls\ILOG.CPLEX.dll</HintPath></Reference>
  <Reference Include="NLog"><HintPath>..\..\dlls\NLog.dll</HintPath></Reference>
  <Reference Include="OptimFoundation.Core"><HintPath>..\..\dlls\OptimFoundation.Core.dll</HintPath></Reference>
  <Reference Include="OptimFoundation.Cplex"><HintPath>..\..\dlls\OptimFoundation.Cplex.dll</HintPath></Reference>
</ItemGroup>
```

- `Generated/` 是 generator 產碼落地供人檢視，MUST `Compile Remove`，否則與編譯期輸出撞名
- DLL 唯一來源是 repo 根 `dlls/`；NEVER 用 NuGet、NEVER 從別處抓 DLL

**✅ 過關條件**：`dotnet build` 成功（此時還沒寫模型，只驗環境）。

---

## §2 資料層 — Set / Parameter / Dataload

轉譯順序固定：**Set → Parameter → Dataload**。Set 與 Parameter 各自從 `IDataSource` 讀進來，`Dataload` 的 ctor 只負責把「讀哪些東西」一行一行寫清楚。

**Dataload 的最終目標只有一個：把數學模型需要的資料整理成可驗證、可直接注入模型的 Set / Parameter 積木。** 它是資料來源與模型積木之間的 adapter，不是資料生成器、不是商業邏輯層，也不是另一個建模層。

### 2.0 CSV-first：先按模型維度整理資料，再讓 Dataload 讀

預設工作方式是先看 Model.md 的 Sets / Parameters，把資料按維度落成 `Data/*.csv`，Dataload 只做顯式載入：

| 模型元素 | CSV 形狀 | 範例 |
| --- | --- | --- |
| `Set_Item` | `Data/Set_Item.csv`；一列一個成員、無表頭 | `ItemA`、`ItemB` |
| `Parameter_Demand{Item,Date}` | `Data/Parameter_Demand.csv`；表頭是 `Item,Date,QTY`，一列是一個維度組合 | `ItemA,2026-01-01,10` |
| scalar `Parameter_ShortagePenalty` | `Data/Parameter_ShortagePenalty.csv`；只有 `QTY` 表頭與一筆值 | `QTY` / `1.0` |
| `Parameter_PreAssign{Item}`（`HasValue=false`） | 表頭只放 key 維度，不放 `QTY` | `ItemA` |

- MUST 先用維度思考：CSV 每個 key 欄對應一個 `[OptDim<Set_X>("Name")]`；數值欄才是 `QTY`
- MUST 讓 CSV 成為模型輸入資料的可檢視事實來源；換一批資料應優先換 CSV，不改 Dataload 與模型 code
- NEVER 因為「用迴圈比較快」就在 Dataload 裡用 `for` / `Enumerable.Range` / `Random` / enum 掃描 / 日期迴圈製造 Set 或 Parameter 資料
- NEVER 在 Dataload 裡補齊、猜測、擴增題目未提供的資料；缺資料應回到 CSV／上游資料準備流程修正
- 大量合成 benchmark、測試資料或可重現樣本若真的需要生成，MUST 放在獨立的資料準備工具或 `gen-csv` 模式，先產出 CSV；正式求解時的 Dataload 仍只讀 CSV
- `LoadInline` 僅限極小教學／單元測試；`LoadFrom` 僅限該 Set 在資料語意上確實只能由既有來源推導。兩者都不是一般專案的預設

### 2.1 Set 積木 — 一顆一個檔

```csharp
// Set/Set_Item.cs
using OptimFoundation.Modeling;

namespace MyProject.Set
{
    [OptSet<string>]
    public partial class Set_Item { }
}
```

```csharp
// Set/Set_Date.cs
using OptimFoundation.Modeling;

namespace MyProject.Set
{
    [OptSet<DateTime>]
    public partial class Set_Date { }
}
```

- MUST 一顆 Set 積木一個 `.cs` 檔，檔名 = 類別名 — NEVER 把多顆塞進 `Sets.cs` — Why: 一檔一顆才能靠檔名直接定位（`Set_Date` 就在 `Set/Set_Date.cs`），與 `Parameter_*` / `Variable*_*` 的慣例一致；集中檔改一顆就動到全部人的 diff
- NEVER 把 attribute 與 class 寫在同一行 — ALWAYS attribute 各自獨立一行 — Why: 加第二個 attribute（`[FullGrid]`、多維 `[OptDim]`）時不必重排，且宣告與修飾一眼分得開
- 合法元素型別：`string` / `DateTime` / `int` / `long` / `double` / `decimal`
- NEVER 寫裸 `[OptSet]` — ALWAYS 顯式 `[OptSet<string>]` — Why: 兩者產碼相同，但顯式讓元素型別在宣告處一眼可見
- NEVER 用裸 `List<string>` 當 Set — Why: 積木化才會被 `DataContext` 註冊進驗證，裸 List 永遠驗不到

`SetBase<T>` 可用成員：`Count` / `this[i]` / `Contains(x)` / **`Load(source, name = null)`（載入 paved path）** / `LoadInline(...)` / `LoadFrom(IEnumerable<T>)` / `LoadCsv(fileName)`；本身是 `IReadOnlyList<T>`，可直接 `foreach` / LINQ / 丟進 `BuildBVs`，不必另存 `List` 視圖。
四道防呆會丟例外：未載入即用、載入後為空、二次載入、成員重複。

### 2.2 Parameter

同樣**一個 Parameter 一個 `.cs` 檔**，attribute 逐行寫。

```csharp
// Parameter/Parameter_Demand.cs — 含值參數：generator 補 Item / Date / QTY + 兩個建構子
using OptimFoundation.Modeling;
using MyProject.Set;

namespace MyProject.Parameter
{
    [OptParam]
    [OptDim<Set_Item>("Item")]
    [OptDim<Set_Date>("Date")]
    public partial class Parameter_Demand { }
}
```

```csharp
// Parameter/Parameter_PreAssign.cs — 純 key 參數（沒有數值，只表示「這組合存在」）
using OptimFoundation.Modeling;
using MyProject.Set;

namespace MyProject.Parameter
{
    [OptParam(HasValue = false)]
    [OptDim<Set_Item>("Item")]
    public partial class Parameter_PreAssign { }
}
```

```csharp
// Parameter/Parameter_ShortagePenalty.cs — 零維 scalar：generator 只補 QTY
using OptimFoundation.Modeling;

namespace MyProject.Parameter
{
    [OptParam]
    public partial class Parameter_ShortagePenalty { }
}
```

| 規則 | 說明 |
| --- | --- |
| 數值欄位名 | **一律 `QTY`**，NEVER `Quantity` / `Amount`（generator 生成的就叫 QTY） |
| property 名 | `[OptDim<TSet>("Name")]` 的字串，PascalCase，NEVER `ALL_CAPS` |
| attribute 順序 | = key 組成順序，改順序 = 改 key |
| 建構物件 | object initializer：`new Parameter_Demand { Item = i, Date = d, QTY = 5 }` |
| 建構子 | **NEVER 自己寫** — 框架用 reflection 讀屬性順序組 key |
| `[FullGrid]` | opt-in，只標「語意上必須全格覆蓋」的參數；MILP 資料多半稀疏，預設 NEVER 加 |

零維 scalar Parameter 沒有 `[OptDim]`，CSV 只有 `QTY` 欄且 MUST 恰好一筆；在 `Program.cs` 組裝時用 `.Single().QTY` 取值，筆數錯誤立即失敗，不用 `FirstOrDefault` 靜默吃掉資料問題。

同一顆 Set 要在同一個類別取兩個角色名（`LotFrom` / `LotTo`）→ 只有 `[OptDim]` 做得到。
逃生口 `[OptParam<Set_A, Set_B>]` 與字串式 `[OptParam("Item", "Date:DateTime")]` 仍合法可編譯，既有 code 不必遷移，但新 code 一律用 `[OptDim]`。

### 2.3 Dataload

```csharp
using OptimFoundation.Core;
using OptimFoundation.Core.IO;
using MyProject.Parameter;

namespace MyProject.Set
{
    public partial class Dataload : DataContext
    {
        public Set_Item ITEM = new();
        public List<Parameter_Demand> parameter_Demand = new();
        public List<Parameter_ShortagePenalty> parameter_ShortagePenalty = new();

        // 無參數建構子只做一件事：把預設來源（CSV）餵給下面那個真正讀檔的 ctor。
        // 有它，Program.cs 才能寫成 OptData.Load(() => new Dataload())；要換來源就改傳 source，讀檔邏輯一行都不用動。
        public Dataload() : this(new CsvDataSource()) { }

        // ctor 就是「讀檔的家」：每行一句、顯式讀，換 CSV / Oracle / 記憶體只換 source
        public Dataload(IDataSource source)
        {
            ITEM.Load(source); // → source.LoadSet("Set_Item")，字串自動依 T 轉型
            parameter_Demand = source.LoadParam<Parameter_Demand>("Parameter_Demand");
            parameter_ShortagePenalty = source.LoadParam<Parameter_ShortagePenalty>("Parameter_ShortagePenalty");
        }
    }
}
```

- MUST `public partial class Dataload : DataContext`（`partial` + 繼承缺一不可）— Why: 少任一個，generator 的註冊碼不會產生，這顆 Dataload 永遠不會被驗證
- MUST 建構走 `OptData.Load(() => new Dataload())` — NEVER 把裸 `new Dataload()` 當終點 — Why: 裸 new 仍可編譯，但完全跳過參照完整性 / 重複 key / 數值 sanity 驗證，壞資料直接進 solver 產出「看起來最佳」的錯答案
- MUST 讓 ctor 保持宣告式：一行載入一顆 Set 或一份 Parameter；讀完後，欄位集合就是模型所需資料積木的完整清單
- MUST 把 penalty、capacity、bound、Big-M 輸入等模型數值也做成 Parameter CSV；scalar parameter 是只有一筆 `QTY` 的零維 Parameter，不是 hardcode public field
- NEVER 在 ctor 寫資料生成邏輯、資料補值規則或模型判斷；這些會把輸入事實藏進 code，導致換資料也必須改 code
- NEVER 手寫驗證邏輯（`ValidateSetsCoverParameters()` 這種）— 機械檢查集中在框架 `DataContext`
- NEVER `try/catch` 吞 `DataValidationException` — 它刻意一次列出全部問題後 fail-fast
- 由資料推導的比值（Big-M 之類）用 `Numeric.SafeRatio(分子, 分母, context: "BigM")`，NEVER 手寫裸除法

**Set 載入方式（預設走 source）**

| 方式 | 寫法 | 何時用 |
| --- | --- | --- |
| **`Load(source)`（預設）** | `ITEM.Load(source);` | **預設一律用這個**；內部就是 `source.LoadSet("Set_Item")`，換 CSV / InMemory 不動這行 |
| `LoadInline` | `ITEM.LoadInline("A", "B");` | 僅限極小教學、單元測試；正式專案仍應落成 CSV |
| `LoadFrom` | `ITEM.LoadFrom(parameter_Demand.Select(p => p.Item).Distinct());` | **逃生口**：這顆 set 真的沒有獨立來源，只能從已載入的 parameter 或外部 query 導出時 |
| DB | `ITEM.LoadFrom(db.LoadSet("SELECT ..."));` | DB 是 query-only（第一引數是 SQL 不是名稱），故不走 `Load(source)` |

- MUST 預設 `ITEM.Load(source)` — NEVER 一律用 `LoadFrom(parameter_X.Select(...).Distinct())` 從參數反推 — Why: 反推出來的 set 只會有「參數表裡出現過」的成員，題目允許但這批資料沒用到的維度會靜默消失，變數少建、約束少建，解出來看起來正常但根本不是原題
- set 名兩種形式等價（`"Item"` / `"Set_Item"`），省略時預設 = 類名去 `Set_` 前綴；CSV 對到 `Data/Set_Item.csv`
- 元素型別非 string（`DateTime` / `int` / `double`…）不必自己 parse，`Load` 會依 `T` 自動轉型

**換資料來源**：`Dataload(IDataSource source)` 這個 ctor 不動，只換傳進去的實作（`CsvDataSource` / `InMemoryDataSource` / 測試替身）。

**✅ 過關條件**：

- `OptData.Load(() => new Dataload())` 不丟例外，且各 Set 的 `Count` 與題目一致
- Model.md 的每顆 Set / Parameter 都能對到一個明確 CSV 與一行載入敘述
- Dataload ctor 只有載入與必要的安全衍生，沒有迴圈造資料、隨機資料、隱藏預設值或題目規則
- 換一批合法 CSV 後，不必修改 Dataload、Objective 或 Constraint code

---

## §3 變數層

一樣**一個變數一個 `.cs` 檔**，attribute 逐行寫。

```csharp
// Variable/VariableB_Assign.cs
using OptimFoundation.Modeling;
using MyProject.Set;

namespace MyProject.Variable
{
    [OptVar]
    [OptDim<Set_Item>("Item")]
    [OptDim<Set_Date>("Date")]
    public partial class VariableB_Assign { }
}
```

**前綴是 load-bearing，不是命名慣例**：

| 前綴 | 型別 | 建立方式（依前綴推型） | 指定 bounds |
| --- | --- | --- | --- |
| `VariableB_` | Binary 0/1 | `BuildVars<T>(sets...)` | `BuildBVs<T>(sets...)` |
| `VariableX_` | Continuous | `BuildVars<T>(sets...)` | `BuildCVs<T>(lb, ub, sets...)` |
| `VariableI_` | Integer | `BuildVars<T>(sets...)` | `BuildIVs<T>(lb, ub, sets...)` |

- 前綴非 `B_`/`X_`/`I_` → compile error `OPTF001`（錯誤訊息會教正確取名）
- NEVER 在 `[OptVar]` 參數指定型別（該參數已移除）
- 0 維純量變數：只掛光桿 `[OptVar]`，不加任何 `[OptDim]`
- 變數名格式：`ClassName@dim1@dim2`，分隔符是 `@`，**NEVER `|`**

**MUST 傳入 sets 的順序 = `[OptDim]` 宣告順序** — Why: 順序錯了不會報錯，只會靜默把索引接錯，最後解出一個「別的題目」的答案。

---

## §4 模型層 — Objective 與 Constraint

### 4.1 Pool API（整個套件的核心手感）

`AddLHS` / `AddRHS` 是「往池子裡丟」，可以反覆呼叫；`CreateXxx` 是「出池」，出完自動清空、接著寫下一條。

```csharp
engine.AddLHS(coef, new VariableX_Produce { Item = i }); // LHS 變數項
engine.AddLHS(constant); // LHS 常數項
engine.AddRHS(coef, new VariableB_Open { Item = i }); // RHS 變數項（框架自動移項）
engine.AddRHS(constant); // RHS 常數

engine.CreateLessEqual($"{ConstraintName}@{i}"); // <=
engine.CreateGreatEqual($"{ConstraintName}@{i}"); // >=
engine.CreateEqual($"{ConstraintName}@{i}"); // ==
engine.CreateRange(lb, ub, $"{ConstraintName}@{i}"); // lb <= LHS <= ub（只吃 LHS）
```

`CreateEqual` / `CreateLessEqual` / `CreateGreatEqual` / `CreateRange`（以及 Phase 3 實驗才會用到的 soft API）會由 `EngineBase` 自動記錄限制式群組與實際建立數量。`ConstraintBase.ConstraintCount` 已 obsolete，NEVER 再手動寫 `ConstraintCount++`。

**天條：Model.md 左邊的項進 `AddLHS`、右邊的項進 `AddRHS`，`>=` 就用 `CreateGreatEqual`。NEVER 自行移項 / 改號 / 翻轉方向 / 合併化簡** — Why: 轉譯必須能逐條對回 Model.md 驗證，動過手腳就驗不了。

### 4.2 一條限制式 = 一個檔

```csharp
using OptimFoundation.Cplex;
using OptimFoundation.Core;
using MyProject.Set;
using MyProject.Parameter;
using MyProject.Variable;

namespace MyProject.Constraint
{
    /// <summary>
    /// ∀ item ∈ ITEM：Σ_date Assign[item][date] ≤ MaxDays[item]
    /// </summary>
    public class Constraint_MaxDays : ConstraintBase
    {
        private readonly Set_Item items;
        private readonly Set_Date dates;
        private readonly List<Parameter_MaxDays> maxDaysByItem;
        private readonly OptEngine engine;

        public Constraint_MaxDays(
            Set_Item items,
            Set_Date dates,
            List<Parameter_MaxDays> maxDaysByItem,
            OptEngine engine)
        {
            this.items = items;
            this.dates = dates;
            this.maxDaysByItem = maxDaysByItem;
            this.engine = engine;
        }

        public void Build()
        {
            foreach (var item in items)
            {
                foreach (var date in dates)
                    engine.AddLHS(1, new VariableB_Assign { Item = item, Date = date });

                var maxDays = maxDaysByItem.FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0;
                engine.AddRHS(maxDays);

                engine.CreateLessEqual($"{ConstraintName}@{item}");
            }
        }
    }
}
```

- 繼承 `ConstraintBase` 取得 `ConstraintName`（= 類別名）；舊的 `ConstraintCount` 已 obsolete，計數改由 `EngineBase` 自動完成
- 限制式命名 MUST `ConstraintName@index1@index2`；`DateTime` 索引用 `{date:yyyy_MM_dd}`
- class 開頭 `<summary>` MUST 貼上 Model.md 對應的數學式 — Why: 驗收時逐條對照靠這行
- Constructor MUST 逐項列出實際依賴的 `Set_*`、`List<Parameter_*>`、scalar 常數與 `OptEngine`；NEVER 接收 `Dataload`，也 NEVER 把暫時用不到的資料一起傳進來
- Objective / Constraint 檔案內不得宣告 `Dataload` field、property 或 constructor parameter；也不要另造 `ModelContext` 之類的整包物件繞過這條規則
- Set / Parameter 集合直接傳框架實際型別；scalar 常數直接按值傳入。依賴新增或刪除時，constructor 簽名與 `Program.cs` call site 必須同步改
- 通常不必自行輸出計數：`engine.Solve()` 會在求解前自動記錄完整 build summary。只有需要自訂診斷格式時，才在 `Program.cs` 的 `.AddModel(...)` pipeline 最後讀 `engine.ConstraintBuildCounts` / `engine.ConstraintCount`；NEVER 放進單一 Constraint class，也不要維護第二份計數

### 4.3 目標式

```csharp
public class ObjectiveFunction
{
    private readonly Set_Item items;
    private readonly double shortagePenalty;
    private readonly OptEngine engine;

    public ObjectiveFunction(Set_Item items, double shortagePenalty, OptEngine engine)
    {
        this.items = items;
        this.shortagePenalty = shortagePenalty;
        this.engine = engine;
    }

    public void Build()
    {
        foreach (var item in items)
            engine.AddLHS(shortagePenalty, new VariableX_Shortage { Item = item });

        engine.CreateMinimize(); // 或 CreateMaximize()
    }
}
```

Objective 與 Constraint 遵守同一條依賴規則：建構子只收這條式子真正使用的資料，NEVER 收整包 `Dataload`。

### 4.4 Phase 2 禁止 soft constraint

正式模型的 `Constraint_*.cs` MUST 只使用 hard constraint API：`CreateLessEqual` / `CreateGreatEqual` / `CreateEqual` / `CreateRange`。

- NEVER 在 Phase 2 或標準 `.AddModel(...)` pipeline 使用 `CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft`
- NEVER 因為模型 infeasible、需求不確定或想讓 solver「比較容易有解」就自行改成 soft；這會改變數學模型，不是實作細節
- Soft constraint 只屬於 Phase 3 的**模型結構實驗**，必須由使用者明確要求，並建立具名的 experiment variant
- 實驗 soft variant 不得覆蓋、取代或混入 canonical hard model；報告結果時必須標出被放鬆的限制式、penalty 與違反量

換句話說：**寫限制式時一律 hard；做實驗時才可能 soft。**

### 4.5 係數來源（天條）

✅ Good

```csharp
var profit = profitByItem.FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0;
engine.AddLHS(profit, new VariableX_Produce { Item = item });
```

❌ Bad

```csharp
engine.AddLHS(8.0, new VariableX_Produce { Item = item }); // 裸數字
engine.AddLHS(profitByItem.First(p => p.Item == item).QTY, new VariableX_Produce {}); // LINQ 內嵌
```

Why: 裸數字讓「改一個係數」變成全專案搜尋；LINQ 內嵌則是中間值 debug 不出來、`First` 沒資料直接炸。
**即使題目只提到兩三個數字，也 MUST 先定義成 Parameter 再經 dataload 取用。**

---

## §5 Program.cs — 唯一組裝點

所有層級關係都直接寫在這一個檔的 pipeline。Solver 設定獨立宣告；變數、目標式、限制式依序直接放入 `.AddVariables(...)` 與 `.AddModel(...)`，NEVER 再包成 `CreateVariables()`、`BuildModel()` 或其他純轉呼叫 helper。

```csharp
if (args.Contains("experiment"))
{
    RunExperiment();
    return;
}

// ── solve 模式 ──
var dataload = OptData.Load(() => new Dataload());
var shortagePenalty = dataload.parameter_ShortagePenalty.Single().QTY;

// ── 求解器設定：在 Program.cs 獨立、明確設定 ──
var config = new CplexConfig
{
    epGap = 0.03,
    timeLimit = 300,
    workThreads = 8,
    enableLog = true,
    exportSol = true,
    exportLP = true,
    exportMPS = true,
};

using (var model = new OptModel("MyProject")
    .UseConfig(() => config)
    // 每個組成各占一個 pipeline step；不要用 block lambda 再包成一組。
    .AddVariables(engine => engine.BuildBVs<VariableB_Assign>(dataload.ITEM, dataload.DATE))
    .AddVariables(engine => engine.BuildCVs<VariableX_Shortage>(dataload.ITEM))
    // 目標式 MUST 先於所有限制式註冊。
    .AddModel(engine => new ObjectiveFunction(dataload.ITEM, shortagePenalty, engine).Build())
    .AddModel(engine => new Constraint_MaxDays(dataload.ITEM, dataload.DATE, dataload.parameter_MaxDays, engine).Build())
    .AddModel(engine => new Constraint_Coverage(dataload.ITEM, dataload.DATE, dataload.parameter_Demand, engine).Build())
    .OnSolved(engine => dataload.WriteToCSV(engine)))
{
    bool ok = model.Execute();
    Logging.Info($"求解結果：{(ok ? "成功" : "失敗")} Status={model.optEngine.Status}");

    if (!ok && model.optEngine.Status == SolveStatus.Infeasible)
    {
        var conflicts = model.optEngine.GetConflictConstraints(); // 框架已自動跑 IIS
        Logging.Info($"衝突限制式（{conflicts.Count}）：{string.Join(", ", conflicts)}");
    }
}

return;
```

| 規則 | 說明 |
| --- | --- |
| `OptModel` 是 composition root | 保證 `AddVariables` 先於 `AddModel`，內建 build / solve 計時與 log |
| Solver 設定獨立宣告 | 在 `Program.cs` 先建立具名 `config`，再 `.UseConfig(() => config)`；NEVER 包成 `NewConfig()` 或藏在 pipeline |
| 每個組成各占一行 pipeline | 每種 `BuildBVs/CVs/IVs` 各寫一行 `.AddVariables(...)`；Objective 與每條 Constraint 各寫一行 `.AddModel(...)`；NEVER 拆成多行參數，也 NEVER 用 block lambda 把多個組成包成一組 |
| 新增一條限制式 | Constraint/ 加一個檔 + `.AddModel(...)` pipeline 加一行，**沒有第三個地方要改** |
| NEVER 在 `.AddModel(...)` 直接寫 `AddLHS` / `AddRHS` | pipeline 只列組成與呼叫順序；數學式仍寫在各 `Objective` / `Constraint_*.cs` |
| NEVER 把 `Dataload` 傳入 Objective / Constraint | pipeline 在 new 類別時逐項傳入它實際使用的 Set / Parameter / scalar；建構子簽名就是依賴清單 |
| NEVER 包裝組裝流程 | 禁止 `VariableCreate`、`CreateVariables()`、`BuildModel` 或其他只負責轉呼叫的類別／函式；review 必須能從 pipeline 直接讀完模型組成 |

### CplexConfig 常用欄位

```csharp
var config = new CplexConfig
{
    epGap = 0.03, // MIP gap 容忍度（3%）；求最佳解設 0.0
    timeLimit = 300, // 秒；null = 無限
    workThreads = 8,
    enableLog = true, // CPLEX log
    exportSol = true, // Sols/*.sol
    exportLP = true, // Models/*.lp（可讀模型，debug 神器）
    exportMPS = true,
};
```

欄位是 **camelCase public field**，直接 `=` 賦值。抽象旋鈕（`Emphasis` / `Seed` / `FeasibilityTol`…，來自 `ITunableConfig`）與 camelCase 欄位指向同一設定，前者跨引擎通用。完整旋鈕表見 `$OPT/AI-Modeling/tuning/CLAUDE.md`。

---

## §6 取解與輸出

```csharp
public void WriteToCSV(OptEngine engine)
{
    Logging.Info($"Status={engine.Status} Obj={engine.GetObjectiveValue():F4} " +
                 $"BestBound={engine.BestObjValue:F4} MIPGap={engine.MIPGap:P2}");

    var assign = engine.GetSetVarValues<VariableB_Assign>(); // Dictionary<完整名, 值>
    double one = engine.GetVariableValue("VariableB_Assign@ItemA@2026-01-01");

    // 軟性違反量：用前綴過濾
    var deficits = engine.GetSolution()
        .Where(kv => kv.Key.StartsWith("Deficit_") && kv.Value > 1e-6).ToList();

    FolderDir.Solution.CreateFolder(); // ★ MUST，否則 WriteSolution 丟 DirectoryNotFoundException
    CsvCtrl.WriteSolution<VariableB_Assign>(engine, "MyProject", "USER");
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

## §7 寫實驗 — 參數掃描

用途：同一個模型套多組 solver 設定，各跑一次，把「完整設定快照 + 收斂數據」記成 `Trial`，累積成 `Experiment` 輸出 `Experiments/<name>.csv` + `.json`（CSV 給人比對，JSON 給後續 LLM 分析）。

本節預設是 solver config 掃描，每個 variant 都必須依照 canonical hard model 的相同順序明確建立。Soft constraint 不屬於一般調參；若使用者明確要求 soft experiment，它是另一個「模型結構實驗」，MUST 使用獨立名稱與獨立 build variant，並保留 hard baseline。可用 API 為 `CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft`，但 NEVER 把它們寫進正式 `Constraint_*.cs` 或標準 `.AddModel(...)` pipeline。

```csharp
static void RunExperiment()
{
    Logging.SetLogFileName("MyProject_Experiment");

    var experiment = new Experiment("myproject-tuning", "掃描 emphasis / gap / threads 的影響");

    var variants = new (string label, Action<CplexConfig> tune)[]
    {
        ("baseline", _ => { }),
        ("emphasis=optimal", c => c.Emphasis = 2),
        ("gap=0.01", c => c.epGap = 0.01),
        ("threads=4", c => c.workThreads = 4),
        ("seed=20260621", c => c.Seed = 20260621),
    };

    int i = 0;
    foreach (var (label, tune) in variants)
    {
        i++;
        // 每個 Trial 的 solver 設定獨立建立，不呼叫 NewConfig() helper。
        var config = new CplexConfig
        {
            epGap = 0.03,
            timeLimit = 60,
            workThreads = 8,
            enableLog = false,
            exportSol = false,
            exportLP = false,
            exportMPS = false,
        };
        tune(config);

        // 每個 Trial 用全新 dataload + engine，狀態不跨 Trial 污染
        var data = OptData.Load(() => new Dataload());
        using var engine = new OptEngine(config);
        engine.Build();

        // 實驗 pipeline 也逐項列出模型組成，不包 CreateVariables() / BuildModel()。
        engine.BuildBVs<VariableB_Assign>(data.ITEM, data.DATE);
        engine.BuildCVs<VariableX_Shortage>(data.ITEM);

        var shortagePenalty = data.parameter_ShortagePenalty.Single().QTY;
        new ObjectiveFunction(data.ITEM, shortagePenalty, engine).Build();
        new Constraint_MaxDays(data.ITEM, data.DATE, data.parameter_MaxDays, engine).Build();
        new Constraint_Coverage(data.ITEM, data.DATE, data.parameter_Demand, engine).Build();

        var trial = Trial.Capture(engine, label, () => engine.Solve());
        experiment.AddTrial(trial);

        var m = trial.Metrics;
        Logging.Info($"[Experiment] ({i}/{variants.Length}) {label}: Status={m.Status} " +
                     $"Obj={m.ObjectiveValue:G6} Gap={m.MipGap:P2} Time={m.RunTimeMs:F0}ms " +
                     $"Nodes={m.NodeCount} Vars={m.VarCount} Cons={m.ConstraintCount}");
    }

    experiment.Save();
    Logging.Info($"[Experiment] 完成：{experiment.Trials.Count} 個 Trial → {FolderDir.Experiment.GetPath()}");
}
```

| 規則 | 說明 |
| --- | --- |
| MUST 每個 variant 全新 `OptData.Load` + 全新 `OptEngine` | 舊 engine 殘留變數池會讓下一組跑到污染的模型 |
| MUST 掃描時 `verbose: false` | solver log 與 LP/MPS 匯出的 I/O 會蓋過要量的時間差 |
| NEVER 用 `CreateVariables()` / `BuildModel()` 包裝 | 每個 variant 在實驗 pipeline 逐項列出變數、目標式、限制式；項目與順序 MUST 對齊 canonical pipeline |
| `Trial.Capture` | 自動偵測 `ITrajectorySource` 啟用收斂軌跡（CPLEX 支援），不接管也不 Dispose engine |
| `Experiment.Save()` | 跨 run 累積，同名實驗以 RunAt + Label 去重 |
| 一次只動一個旋鈕 | 同時改兩個就分不出是哪個造成差異 |

**MUST 先過 §8 正確性 gate 才做 tuning。** 對錯的模型調參數只會更快得到錯答案。

---

## §8 驗收 — 解驗證協定

`dotnet run` 有輸出 ≠ 會動。MUST 依序四步，任一步不過就停下回報，NEVER 宣稱完成：

1. **Status 三分診斷**
   - `Optimal` → 進第 2 步
   - `Infeasible` → 讀 `IISs/*.ilp` 或 `GetConflictConstraints()` 拿最小衝突集；先自查 Big-M 是否太小、有無互斥硬約束
   - `Unbounded` → 某個方向漏了界；查該變數 UB 或漏掉的上限約束
2. **可行性代回**：把解值代回**每一條** constraint，確認 `LHS op RHS` 成立 — Why: solver 回 Optimal 只保證「它解的那個模型」可行，不保證那個模型 = 你的題目
3. **單位一致性**：目標值與關鍵變數的單位跟題目一致（利潤 = 錢、產量 = 件、工時 = 時），量級合理
4. **LP bound sanity**：max 問題整數解 ≤ LP relaxation bound；min 問題 ≥。差太離譜 → 疑 Big-M 或係數錯

✅ Good：四步都過，且目標值對得上 Model.md 的手算小例
❌ Bad：看到 `Status == Optimal` 就回報「解出來了」

build 失敗走 fix loop：擷取 compiler error → 修 → 重 build，**至多 5 次**；仍失敗停下回報，NEVER 硬掰。

---

## §9 API 速查卡

```csharp
// 建構
var config = new CplexConfig { epGap = 0.0, timeLimit = 60, workThreads = 4, enableLog = true };
var engine = new OptEngine(config);
engine.Build();

// 建變數
engine.BuildBVs<VarB>(setA, setB); // Binary
engine.BuildCVs<VarX>(setA); // Continuous [0, 1e100]
engine.BuildCVs<VarX>(0, 100, setA); // Continuous [0, 100]
engine.BuildIVs<VarI>(0, 10, setA); // Integer [0, 10]

// 建限制式
engine.AddLHS(coef, new VarX { A = a });
engine.AddLHS(constant);
engine.AddRHS(coef, new VarB { A = a });
engine.AddRHS(constant);
engine.CreateLessEqual($"{ConstraintName}@{a}");
engine.CreateGreatEqual($"{ConstraintName}@{a}");
engine.CreateEqual($"{ConstraintName}@{a}");
engine.CreateRange(lb, ub, $"{ConstraintName}@{a}");
engine.CreateLeSoft(rhs, penalty); // EXPERIMENT ONLY：禁止出現在 canonical Constraint / .AddModel pipeline
engine.CreateGeSoft(rhs, penalty); // EXPERIMENT ONLY
engine.CreateEqSoft(rhs, penalty, name); // EXPERIMENT ONLY

// 目標式
engine.AddLHS(coef, new VarX { A = a });
engine.CreateMinimize(); // 或 CreateMaximize()

// 求解與取解
bool ok = engine.Solve();
double obj = engine.GetObjectiveValue();
var dict = engine.GetSetVarValues<VarX>();
var m = engine.LastMetrics; // gap / bound / node·iter / 軌跡

// 輸出
FolderDir.Solution.CreateFolder();
CsvCtrl.WriteSolution<VarX>(engine, "DataId", "User");

// 日誌
Logging.SetLogFileName("MyProject");
Logging.Info("訊息");

// 實驗
var exp = new Experiment("my-tuning", "說明");
exp.AddTrial(Trial.Capture(engine, "label", () => engine.Solve()));
exp.Save();
```

### 黑名單 — 這些方法不存在，寫了必爛

| ❌ 錯誤呼叫 | ✅ 正確替代 |
| --- | --- |
| `engine.GetVarSol(name)` | `engine.GetVariableValue(name)` |
| `engine.GetSetVarSol<T>()` | `engine.GetSetVarValues<T>()` |
| `CSVCtrl.SaveToCSV<T>(...)` | `CsvCtrl.WriteSolution<T>(engine, dataId, userId)` |
| `CSVCtrl.xxx`（大寫 V） | `CsvCtrl.xxx` |
| `FolderDir.Result` | `FolderDir.Solution` |
| `new OptEngineConfig { ... }` | `new CplexConfig { ... }` |
| `engine.AddPool` / `AddPoolRHS` | `engine.AddLHS` / `AddRHS` |
| `BuildBVs(typeof(T), sets)` | `BuildBVs<T>(sets)` |
| 假設變數名分隔符是 `\|` | 是 `@` |
| 假設 `ProjFolder` 建構子會建資料夾 | MUST 手動 `FolderDir.Solution.CreateFolder()` |
| `override Build()` / `override Solve()` | 已非 virtual；改覆寫 `BuildCore()` / `SolveCore()` |

---

## §10 常見錯誤

| 症狀 | 真正原因 | 修法 |
| --- | --- | --- |
| `CS0246` 找不到型別 | csproj `HintPath` 層數錯 | `Projects/` 底下一律 `..\..\dlls\` |
| generator 沒產碼 | 類別漏了 `partial`，或 `Set_*` 漏掛 `[OptSet<T>]` | 補上；漏掛且被 Dataload 引用會報 `OPTF006` |
| `OPTF001` | 變數前綴不是 `VariableB_/X_/I_` | 依前綴表改名 |
| `DirectoryNotFoundException` | 寫檔前沒 `CreateFolder()` | `FolderDir.Solution.CreateFolder()` |
| `CS8618`（手寫 class 時） | string property 沒初始化 | `public string Item { get; set; } = string.Empty;` |
| 解出來但數值離譜 | 變數 sets 傳入順序與 `[OptDim]` 不一致 | 逐一對齊順序 |
| Infeasible 但模型看起來對 | Big-M 太小、或兩條硬約束互斥 | 讀 `IISs/*.ilp`；`exportLP = true` 開 `.lp` 檔用肉眼查 |
| 改一個係數要改好幾個檔 | 係數 hardcode 在 Constraint 裡 | 全部移進 Parameter 的 `QTY` |
| 換資料還要改 Dataload code | Dataload 內用迴圈／Random／enum／日期邏輯生成資料 | 把資料按 Set / Parameter 維度落成 CSV，Dataload 改回只讀 source |

### 反模式

❌ `try { ... } catch (Exception) { throw; }` — 什麼都沒做卻讓每個方法多兩層縮排，直接刪掉
❌ 在 Variable / Parameter 類別寫建構子 — 框架用 reflection 組 key，自己寫只會干擾
❌ Objective / Constraint 建構子接收 `Dataload` — 改成只注入該式子使用的 Set / Parameter / scalar 與 `OptEngine`
❌ 在 Constraint class 手動 `ConstraintCount++` — `EngineBase` 已由每次 `CreateXxx` 自動統計，舊 property 已 obsolete
❌ 用 `VariableCreate`、`CreateVariables()`、`BuildModel` 或其他 helper 包裝模型組裝 — 每個組成必須直接列在 `Program.cs` 的 `.AddVariables(...)` / `.AddModel(...)` pipeline
❌ 用 `NewConfig()` helper 或把 solver 設定藏在 `.UseConfig(...)` — 在 `Program.cs` 獨立建立具名 `CplexConfig` 再傳入
❌ 在正式 `Constraint_*.cs` 或標準 `.AddModel(...)` pipeline 使用 `CreateLeSoft` / `CreateGeSoft` / `CreateEqSoft` — soft constraint 僅供使用者明確要求的 Phase 3 結構實驗
❌ 在 Dataload ctor 用迴圈、`Random`、enum 或日期運算製造模型輸入 — 移到獨立資料準備流程先產 CSV；Dataload 只負責讀成積木
❌ 改 OptimFoundation 框架本體（`dlls/` 是唯讀的）— 缺方法就在專案端寫 helper / extension

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
| `$OPT/AI-Modeling/Template/` | 可直接複製的起始範本 |
| `$OPT/AI-Modeling/tuning/CLAUDE.md` | CplexConfig 全旋鈕對照表 |
