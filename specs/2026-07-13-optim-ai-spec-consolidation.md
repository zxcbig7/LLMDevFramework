# 規格：OptimizationFramework AI 規範整併（AI-Modeling + OptimFoundation + MILP domain）

status: approved（需求已由 Vic 逐項確認，2026-07-13）
executor: Opus（本文件為唯一執行依據）
scope: 三處 —— `$OPT/AI-Modeling`、`$OPT/OptimFoundation`、`$FW/MILP Model`（$OPT = `C:/Users/zxcbi/Desktop/Projects/OptimizationFramework`、$FW = LLMDevFramework repo 根）

---

## 一、已驗證的缺陷清單（修正對象）

### OptimFoundation

| # | 缺陷 | 證據 |
|---|---|---|
| A1 | CLAUDE.md 與 CodeMap.md 寫「全鏈 net48」「CPLEX 只支援 .NET Framework」，實際 10 個 csproj 全是 `net8.0`（Generators 為 netstandard2.0） | `CLAUDE.md:4,10`、`CodeMap.md:65-74,112` vs 全部 csproj 實測 |
| A2 | 「9 專案 / 45 tests」過時：實際 csproj 10 個（Templates 多了 `FJSP_BASIC`）、tests 內 `[Fact]/[Theory]` 61 個（最近實跑 69 passed） | `CLAUDE.md:19,26`、`CodeMap.md:21,64,70` |
| A3 | `.claude/` 空資料夾，無 hook / settings | 實測 |
| A4 | 「Generators 尚無消費者」過時 —— AI-Modeling 的 HospitalRostering_Generator 已消費 | `CLAUDE.md:24` vs 消費端 `Generated/` 實測 |

### AI-Modeling

| # | 缺陷 | 證據 |
|---|---|---|
| B1 | `automated/CLAUDE.md`（541 行）描述不存在的 ASP.NET Web API：`src/AIModeling.Api/`、RAG、Semantic Kernel、lowercase `projects/`、`docs/` 都不在 repo | `automated/CLAUDE.md:5,361-401,500-532` vs 目錄實測 |
| B2 | automated 文件內含 interactive 流程（「Framework 1」段：Stage 4/7b 暫停確認），與「全自動不需人在迴路」的定位矛盾 | `automated/CLAUDE.md:405-470` vs `AGENTS.md:13` |
| B3 | Projects/ 兩套不相容結構並存：`MaxWeightIndependentSet`（stages/+csharp/）vs `HospitalRostering_Generator`（扁平），無規則定何時用哪套 | 目錄實測 |
| B4 | 宣稱「單一來源在 AGENTS.md」，但「NEVER 改框架本體」「NEVER 發明 API」兩條天條只在 CLAUDE.md | `CLAUDE.md:3,29-34` vs `AGENTS.md:26-31` |
| B5 | tuning 規範散在 4 處且資料夾拼錯：`truning/`、`interactive/phase-3-tuning.md`、`automated/CLAUDE.md` Stage 15 段、`automated/specs/2026-06-21-model-tuning-protocol.md` | 實測 |
| B6 | API 對照表兩份無同步規則：`CPLEX_API_REFERENCE.md`（1237 行複本，天條指定權威）vs 框架 `specs/developer-guide.md`（活源頭） | 兩檔實測 |
| B7 | **AGENTS.md 指定的「可運作範例」HospitalRostering_Generator build 失敗（42 errors）**：csproj `ProjectReference` 指向不存在的 `..\OptimModeling.Generators\`；註解殘留已刪除的 `ClaudeAIAssistant\dlls\`；`Generated/` 下同時有 OptimModeling / OptimFoundation 兩套輸出 | `dotnet build` 實跑 + csproj 實測 |
| B8 | **Generator（Analyzer）引用無規範，現存三種寫法全壞**：① HospitalRostering_Generator → 不存在的 `..\OptimModeling.Generators\`（即 B7）② HospitalRosteringProblem_new → **絕對路徑** `C:\Users\zxcbi\...\AI-Modeling\OptimFoundation\Templates\dlls\...`（違反 NEVER 絕對路徑天條，且路徑不存在）③ Template_CPLEX → 不存在的 `..\Projects\OptimModeling.Generators\`。一般 DLL 引用（Core/Cplex/ILOG/NLog）九專案全部正確指 `dlls/`——規則缺口只在 Analyzer | 全 csproj grep 實測 |
| B9 | **dlls/ 同步時機無規則**：OptimFoundation rebuild 後沒有任何規範要求回填 `dlls/`。事故史：stale Core.dll 遮住 API drift（IO namespace 搬移後 8 個消費端其實編不過，被舊 DLL 掩蓋，2026-07-11 發現） | memory `optimfoundation-net8-validated` |

### LLMDevFramework

| # | 缺陷 | 證據 |
|---|---|---|
| C1 | `MILP Model/CLAUDE.md` 殘留已失效的 ClaudeAIAssistant 路徑引用；且引用的兩 repo 結構即將因本規格改變 | memory `milp-domain-claudeaiassistant-stale` + git status 未 commit 修改 |

---

## 二、已拍板的需求決策（11 項）

| # | 決策 | 內容 |
|---|---|---|
| D1 | **TFM 基準 = net8.0 唯一** | 天條翻轉為「全鏈 net8.0，Generators 維持 netstandard2.0（Roslyn 要求）」；刪除「CPLEX 只支援 .NET Framework」過時說法 |
| D2 | **Web API 已放棄，全刪** | automated/CLAUDE.md 刪除 Web API / RAG / Semantic Kernel / PromptLibrary 所有段落；automated 路線 = `Prompts/` 模板 + Claude Code 驅動 |
| D3 | **專案結構統一扁平** | 廢除 stages/+csharp/ 兩層結構；全部統一 `Model/ Set/ Parameter/ Variable/ Constraint/ Objective/ + Program.cs + csproj`（六夾，Set/ 保留） |
| D4 | **天條唯一權威 = AGENTS.md** | 所有天條收進 AGENTS.md（含目前只在 CLAUDE.md 的兩條）；CLAUDE.md 縮成純 router：導引 + 一行「天條見 AGENTS.md」 |
| D5 | **Set 模組化 = [OptSet] 框架擴充** | OptimFoundation 新增 `SetBase` + AutoSetsGenerator 支援 `[OptSet]`，一 set 一檔、資料來源（inline/CSV/Oracle）可插拔；Dataload 瘦身為純組合器。屬新框架功能，MUST 走 /sdd 在 OptimFoundation 開獨立規格再實作 |
| D6 | **resume = 直寫終點 + status.json** | automated 每站產物直接寫入最終扁平資料夾（數學模型 → `Model/<Name>_Model.md`，code → 對應資料夾）；前段純文字產物（00-03）併成 Model/ 下一份推導文件；`status.json` 留專案根追蹤 completed/current |
| D7 | **tuning 唯一權威 = 根目錄 `tuning/`** | `truning/` 改名 `tuning/`，258 行策略庫為唯一權威；interactive phase-3 / automated / specs 三處改為一行指向，不重複內容 |
| D8 | **API 同步 = 源頭+鏡像+同步天條** | `developer-guide.md` 為源頭；`CPLEX_API_REFERENCE.md` 為鏡像（檔頭標注源頭位置+同步日期）；OptimFoundation AGENTS/CLAUDE 加天條「改 public API 必同步更新 sibling 鏡像」 |
| D9 | **AML 字眼全面廢除** | 唯一術語 =「Model」。Prompts 檔名（`04_AMLModel.md`→`04_Model.md`、`04b_AMLModelVerify.md`→`04b_ModelVerify.md`）+ 所有規範內文 + claudemdTemplate 一次改完；舊專案歷史產物（如 stages/04_aml.md）不回溯 |
| D10 | **OptimFoundation 補 .claude/ 基本盤** | 比照 AI-Modeling：dotnet-build-summary hook + settings.json |
| D11 | **一併涵蓋 $FW/MILP Model domain** | 引用路徑、術語（廢 AML）、新專案結構三處一次改齊 |
| D12 | **DLL 引用規則（收進 AGENTS.md 天條區）** | 見下方「DLL 引用規則全文」 |

### D5 — Set 積木概念：預期結果（/sdd 規格的靶心）

核心比喻：Set 從 Dataload 的散裝欄位變成標準積木；Dataload 變成底板，只負責扣積木。與 [OptVar]/[OptParam] 完全對稱。

寫法目標：

```csharp
// Set/Set_Employee.cs — 一塊積木一個檔，2 行
[OptSet<string>]                      // 元素型別 MUST 顯式寫出（裸 [OptSet] 雖合法，非預設寫法）
public partial class Set_Employee { }

// Model/Dataload.cs — 底板
public class Dataload {
    public Set_Employee EMPLOYEE = new();
    public Dataload() { EMPLOYEE.LoadInline("A","B","C"); }  // 或 LoadCsv/LoadDb/LoadRange
}
```

Constraint 端不變：`foreach (var e in dataload.EMPLOYEE)` 照寫（SetBase 實作 `IEnumerable<T>`）。

驗收式預期結果：

1. 一個 set = 一檔 2 行宣告，body 全由 generator 生成
2. 資料來源（inline/CSV/Oracle/程式生成）切換只改 Load 呼叫，模型與 constraint code 零改動
3. 錯誤左移：`Set_` 命名天條違反 → compile error（沿用 OPTF 診斷機制）；未載入就使用 → 明確例外，NEVER 空集合靜默解出退化解
4. Dataload 只剩組合 + 載入；罰分權重歸 Parameter、CSV I/O 歸框架
5. 常用積木（時間軸/機台/班別）單檔複製即跨專案重用
6. AI 管線的產 Set stage 變機械式宣告，prompt 縮短、LLM 出錯面縮小
7. **維度引用積木（主設計，Vic 已確認方向）**：三個 attribute 全走 **C# 11 泛型**統一語法——`[OptSet<T>]` 宣告元素型別、`[OptParam<>]`/`[OptVar<>]` 直接引用 Set 積木，property 名與元素型別都從積木自動抓 —— (名字,型別) 全系統只在 Set 積木宣告一次

```csharp
[OptSet<DateTime>]                                       // 元素型別直接吃泛型參數
public partial class Set_Date { }                        // (名字,型別) 唯一宣告處

[OptSet<string>]                                         // 元素型別顯式寫出（裸 [OptSet] 合法但非預設）
public partial class Set_Employee { }

[OptVar<Set_Date, Set_Employee>]
public partial class VariableB_ShiftAssign { }           // 生成 DateTime Date + string Employee
```

設計依據：C# attribute 引數不能放 tuple 也不能放裸型別名。paved path = 泛型 attribute（net8 + LangVersion latest 已支援）：Param/Var 配 `where T : ISetBrick` constraint 讓「引用非積木」變 CS0311 原生 compile error；OptSet 的元素型別封閉域由 generator OPTF 診斷把守（C# 無 union constraint）；arity 宣告 `<T1>`…`<T1..T6>`。非泛型 `typeof`/字串版留作逃生口（alias + 舊寫法漸進遷移）。細節全文 → `OptimFoundation/specs/2026-07-13-optset-basic-objects.md`。

**成員名稱推導規則（全機械，唯一命名動作 = 幫積木取名）：**

| 位置 | 規則 |
| --- | --- |
| 積木 class 名 | `Set_<PascalName>`，違反 → OPTF 診斷 |
| Var/Param property 名 | 去 `Set_` 前綴，不轉換（`Set_TruckFleet` → `TruckFleet`）——已驗證與現行 generator 輸出（PascalCase）一致 |
| property 順序 | = attribute 引數順序 → 決定 `InitClassBySets` 的變數 key 組成順序（`ClassName@val1@val2`），**調換順序 = 不同變數，天條寫明** |
| 同 set 雙維度 | alias 直接當 property 名（`From` / `To`） |
| `QTY` | Param 固定生成、永遠最後（既有天條不變） |
| Dataload 欄位名 | UPPER_SNAKE（Pascal 機械轉換，沿用 `dataload.EMPLOYEE` 慣例） |
| CSV 欄頭 / Oracle bind | 框架既有行為（property 名 `.ToUpper()`），不另定義 |

連帶修正：`automated/CLAUDE.md` 舊規則「Set 屬性全大寫+底線（`PRODUCT_INDEX`）」與 generator 實際輸出（PascalCase）矛盾——新規則以 generator 現實為準，舊文件規則作廢（併入 Phase 1 第 3 點重寫）。

**SetBase 契約草案（已對現有框架驗證）：**

- `SetBase<T> : IEnumerable<T>` —— 已驗證直接相容 `Build*Vs(params object[])`（`ConvertSetsToStringLists` pattern match 吃 `IEnumerable<T>`，`VariableBuilder.cs:79-102`）與 constraint `foreach`，框架消費端零改動；DateTime key 序列化沿用框架既定 `yyyy-MM-dd`
- 內部 `List<T>`（保序，key 順序穩定）+ `HashSet<T>` 陪跑（`Contains` O(1)，稀疏 constraint 用）
- **未載入就 enumerate → `InvalidOperationException`**（預期結果 #3 落地）；載入後空集合 → 例外
- **載入即封存**：二次載入 → 例外；重複成員 → 例外（重複會生出重複變數 key）
- 載入 API：`LoadInline(params T[])` / `LoadFrom(IEnumerable<T>)` / `LoadCsv` / `LoadDb`；型別特化（如 `SetBase<DateTime>` 的 `LoadRange`）用 extension method，不進 generator
- generator 對 `[OptSet(...)]` 只生成一行 body：`: SetBase<對應元素型別>`

**資料讀取三檔位（包既有 API，不發明新輪子——已查證）：**

| 檔位 | Set 積木 | Param |
| --- | --- | --- |
| inline | `LoadInline(...)` / `LoadRange(...)` | `list.Add(new Param_X(...))` |
| CSV | `LoadCsv(name)` 內部依 `T` dispatch 到 `CsvCtrl.ReadStrSet/ReadDateSet/ReadIntSet` | `CsvCtrl.BuildParameter<T>()`（既有，表頭按名對位） |
| Oracle | `LoadDb(DbDataSource)` 內部呼叫 `ReadSet(name)`，name 從積木類名推導 | `DbDataSource.ReadParameters<T>()`（既有） |

三檔位切換只動 Dataload 建構子，`Set/ Parameter/ Variable/ Constraint/ Objective/` 全部零改動。property 名 = CSV 欄名（`.ToUpper()`）= Oracle 欄名——命名推導鏈打通讀取層。

/sdd 剩餘待決：要不要 `Reload`（實驗換資料集場景）、**同 set 雙維度的 alias 語法**（from/to 都是 `Set_City` 會撞 property 名，逃生口如 `"From:Set_City"`）、衍生積木（子集/稀疏笛卡兒積）進不進 v1、舊 `List<string>` 與字串維度寫法的遷移路徑。

### D12 — DLL 引用規則全文（AI-Modeling 消費端）

1. **一般組件**（OptimFoundation.Core / Cplex、ILOG.Concert / ILOG.CPLEX、NLog）：`<Reference>` + `HintPath` 相對路徑指 repo 根 `dlls/`（唯一來源）—— 現況已合規，規則明文化即可
2. **Source generator**：統一唯一寫法 `<Analyzer Include="..\..\dlls\OptimFoundation.Generators.dll" />`（依巢狀深度調整 `..` 數）—— NEVER `ProjectReference` 跨 repo、NEVER 絕對路徑、NEVER 指向 OptimFoundation 的 bin 輸出
3. **dlls/ 同步時機**：OptimFoundation 有 public API 變更或 rebuild 後，MUST 在 AI-Modeling 跑 `scripts/setup-dlls.ps1 -Build` 回填 —— Why: stale DLL 會遮住 API drift，消費端看似 build 綠實則已編不過（2026-07-11 事故）
4. **staleness 可偵測**：setup-dlls.ps1 回填時在 `dlls/` 寫入 build 時戳/來源 commit（如 `dlls/VERSION.txt`），供 harness/人工比對（Opus 實作項）
5. OptimFoundation 側規則不變：`$(CplexDir)` / `$(GUROBI_HOME)`、內部 `ProjectReference`、NEVER commit DLL（現況已合規）

---

## 三、執行計畫（Opus 依序執行，每 phase 結尾驗收）

> **執行順序更正（Vic 2026-07-14）**：底層 API 開發（OptSet 框架）MUST 先於一切 AI 治理文件。
> Why: 治理文件（新專案結構 D3、AML→Model D9、模板、命名規則）全都在描述「怎麼用框架 API 寫模型」；
> OptSet 一改 `[OptVar<Set_X>]` 語法，先寫的治理文件就得整批重寫一次。先蓋底層、API 定版，治理再反映最終形。
> 故順序：Phase 0 事實同步（已完成）→ **Phase 1 = OptSet 底層（原 Phase 2）** → Phase 2 = AI-Modeling 治理（原 Phase 1）→ Phase 3 = MILP domain。

### Phase 0 — OptimFoundation 事實同步（低風險，已完成 2026-07-14）

1. `CLAUDE.md`：net48 → net8.0 天條翻轉（D1）；專案數/Templates 清單補 FJSP_BASIC；Generators 註記消費者（A4）；加 API 鏡像同步天條（D8）—— ✅ done
2. `CodeMap.md`：TFM / 專案數 / 測試數全面同步（實測 73 tests、10 專案）—— ✅ done
3. 補 `.claude/`：settings.json + dotnet-build-summary hook（D10）—— ✅ done

驗收：CLAUDE.md/CodeMap.md 無任何 net48 字眼（Generators netstandard2.0 例外說明除外）；`dotnet build` 全綠（實測 0 error）；verifier 驗收（loop 補跑）。

### Phase 1 — OptSet 底層框架（D5，最高優先，先於治理）

> 設計靶心 design-approved：`OptimFoundation/specs/2026-07-13-optset-basic-objects.md`（`[OptSet<T>]`/`[OptParam<>]`/`[OptVar<>]` 泛型統一語法、SetBase 契約、命名推導、三檔位讀取、消費端範例）。Vic 已於對話逐項確認設計並授權 loop 自主實作；開放項用保守預設就地定案並在該檔註記。

1. **Core**：新增 `SetBase<T> : IEnumerable<T>`（List 保序 + HashSet O(1)、四道防呆）+ marker interface `ISetBrick`
2. **Generators 擴充 AutoSetsGenerator**：
   - 泛型 `[OptSet<T>]` → 生成 `: SetBase<T>`；元素型別封閉域用 OPTF 診斷把守（C# 無 union constraint）
   - 泛型 `[OptParam<...>]` / `[OptVar<...>]` → semantic model 解析 Set 積木、抓名稱+型別生成 property（key 順序 = 泛型參數順序）
   - 保留舊字串式 `[OptVar("Date:DateTime")]` 為逃生口（遷移用）
   - `where T : ISetBrick` constraint → 引用非積木 = CS0311 原生錯誤
3. **tests**：SetBase 四道防呆 + generator 生成正確性（新增 xUnit case）
4. **回歸驗證**：`dotnet build` sln 全綠 + `dotnet test` 全綠（含新測試）
5. **文件同步**：`developer-guide.md` 新增 OptSet 章（本設計檔為底稿）→ 同步 `../AI-Modeling/CPLEX_API_REFERENCE.md` 鏡像（D8 首次演練，檔頭標同步日期）
6. **DLL 回填**：跑 `AI-Modeling/scripts/setup-dlls.ps1 -Build`（D12.3 首次演練，含 VERSION.txt 時戳）

驗收：`dotnet test` 全綠且含 OptSet 新測試；一個消費端專案實際用 `[OptSet<T>]`/`[OptParam<>]` build 綠；鏡像檔頭有同步日期。

### Phase 2 — AI-Modeling 治理重構（依賴 Phase 1，API 定版後才做）

1. **AGENTS.md**：收齊全部天條（含 CLAUDE.md 獨有兩條）成唯一權威（D4）；「可運作範例」指引待 B7 修復後確認仍成立
2. **CLAUDE.md**：縮成純 router（D4）
3. **automated/CLAUDE.md 重寫**（D2+B2）：只留 16-stage 定義、stage 輸出對應（依 D3/D6 新結構）、命名規則（用最終 `[OptSet<T>]` 語法 + AML→Model 術語 D9）、LHS/RHS 鐵則、CplexConfig 旋鈕表、resume 協定；刪 Web API/RAG/「Framework 1」段
4. **truning/ → tuning/**（D7）：改名 + 三處指向
5. **AML → Model 全面改名**（D9）：Prompts/ 17 檔檔名與內文、claudemdTemplate/、interactive/ 各檔
6. **Projects/CLAUDE.md**：寫死統一扁平結構（D3）+ resume 規則（D6）
7. **修 B7/B8 — DLL 引用規則落地（D12）**：三個壞掉的 generator 引用（HospitalRostering_Generator、HospitalRosteringProblem_new、Template_CPLEX）統一改為 `<Analyzer Include="..\..\dlls\OptimFoundation.Generators.dll" />` 寫法；清 OptimModeling 殘骸（Generated/ 舊輸出、註解）與 ClaudeAIAssistant 殘留註解；D12 規則寫進 AGENTS.md；受影響專案 MUST `dotnet build` 轉綠
8. **遷移 MaxWeightIndependentSet** 成扁平結構（csharp/ 內容上提、stages/ 精華併入 Model/、status.json 保留）；MUST build 綠

驗收：全部 9 個 Projects + Template_CPLEX `dotnet build` 全綠；**治理面 + 程式碼**（CLAUDE.md/AGENTS.md/Prompts/claudemdTemplate/Projects/Template_CPLEX/README/CPLEX_API_REFERENCE/CodeMap + `.cs`/`.csproj`）grep 無 `AML`、無 `truning`、無 `OptimModeling`、無 `ClaudeAIAssistant`、csproj 無 `C:\` 絕對路徑；天條（含 D12 DLL 規則）只在 AGENTS.md 一處全量出現。

> **範圍註（執行時定案 2026-07-14）**：`tutorial/`（human-facing 教學/HTML slides，另有 tech-doc-story-loop 工作流負責）與 `specs/`、`automated/specs/`（point-in-time 設計記錄，改寫等於竄改歷史）**不在本次治理清理範圍**——這些檔的 AML/OptimModeling/ClaudeAIAssistant 保留。歷史 `stages/` 產物同理（MaxWeightIndependentSet 已於 2-8 遷移刪除，其餘無）。

### Phase 3 — $FW/MILP Model domain 同步（D11，最後做）

1. `MILP Model/CLAUDE.md` 與子規範：失效路徑清除（C1）、術語廢 AML、新專案結構對齊 D3
2. `workflow skill/`（milp-dev）引用同步
3. MILP Model 有 evals/ → MUST 跑 `/harness-eval` 驗證行為無 regression
4. root `CodeMap.md` File Index 行數同步（$FW 根 CLAUDE.md 的既有義務）

驗收：`/harness-eval` PASS；grep $FW 無 ClaudeAIAssistant / AML 殘留。

---

## 四、通用鐵則（執行全程適用）

- NEVER 動數學語意：本次全部是文件/結構/引用重構，不改任何模型的移項/符號/數值
- 每個 phase 結束 MUST 派 verifier fresh-context 驗收後才進下一 phase
- 檔案改動遵守 $FW 鐵則：無欄位對齊、無 BOM、CLAUDE.md ≤ 200 行
- 計數類事實（測試數、專案數）MUST 執行當下實測，不抄本規格
