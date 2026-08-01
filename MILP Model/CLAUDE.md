# MILP Model — 數學模型開發規範（domain root）

<system_context>
MILP / LP / IP 數學模型開發 domain，服務 OptimizationFramework 專案（OptimFoundation CPLEX C# 框架）。
開發 = 三階段 phase gate：**Model Design（建模）→ Foundation Coding（轉譯實作）→ Foundation Tuning（調校）**。
本檔是天條層（硬規則）；各 phase 細節在子資料夾 CLAUDE.md。
外部專案根：`C:/Users/zxcbi/Desktop/Projects/OptimizationFramework`（下稱 `$OPT`）。
</system_context>

<critical_notes>

## 三階段 Phase Gate（天條）

- MUST 依序走三階段：Modeling → Coding →（使用者提出才做）Tuning
- NEVER 在使用者明確確認數學模型前產生任何 `.cs` 檔 —— ALWAYS 先產 `Model/<Project>_Model.md` 等「模型確認」或「開始實作」—— Why: 模型錯了 code 全部重寫，先鎖模型再轉譯是最省的路
- NEVER 對不清楚的術語或題目描述自行猜測 —— ALWAYS 追問後才繼續，確認後補進 `Model/Glossary.md`
- MUST Coding 階段是 Model.md 的**純機械轉譯**，不允許任何自行詮釋；發現 Model.md 有歧義 → 立即停止，回 Model Design 補模型

## 命名（天條）

- NEVER 在 Model.md 用無意義單一字母符號（`i`、`j`、`k`、`x`、`y`、`z`、`t`）—— ALWAYS 用語意名稱（`GlassType`、`Assign_{Employee,Date}`）
- MUST 程式類別名直接對應 Model.md 符號：`Parameter_` / `VariableB_`（binary）/ `VariableX_`（continuous）/ `VariableI_`（integer）/ `Constraint_` 前綴 + 符號語意核心轉 PascalCase
- ★ 變數前綴是 load-bearing（決定型別，非僅慣例）：`VariableB_`→Binary / `VariableX_`→Continuous / `VariableI_`→Integer。generator 依前綴判型並產碼，前綴非法直接 compile error（`OPTF001`，訊息教正確取名）；`BuildVars<T>` 亦由前綴推型 —— NEVER 用 `[OptVar]` 參數指定型別（該參數已移除，attribute 只帶 sets）—— Why: 型別過去在 attribute / 前綴 / Build call 三處各表述一次可互相矛盾，收斂成前綴單一真相後才驗得了
- Set 成員字串 PascalCase 單數：`"Truck"` ✅、`"truck"` ❌、`"Trucks"` ❌

## 數學一致性（天條）

- NEVER 移項、改號、翻轉比較方向、合併化簡 —— ALWAYS Model 左側項 → `AddLHS(...)`、右側項 → `AddRHS(...)`，`>=` → `CreateGreatEqual`、`<=` → `CreateLessEqual`、`=` → `CreateEqual` —— Why: 轉譯必須可逐條對照 Model.md 驗證，動過手腳就驗不了
- MUST 數值保真：所有數值與題目描述完全一致，NEVER 四捨五入、推算、填佔位符
- ★ 禁止 Hardcode：模型所有數值（係數、容量、比例）一律定義在 `Parameter` 類別的 `QTY` 欄位、經 `Dataload` 取得；`Constraint` / `Objective` 內不得出現任何裸數字

## Tuning Gate（天條）

- MUST 先過正確性 gate 才做效能 tuning：`dotnet build` → `Solve()` → `Status == Optimal` → 解 / 目標值對照題目描述驗證
- Tuning 順序：solver 參數（`CplexConfig`）→ IIS / soft constraint → 模型結構；影響模型語意的變更 MUST 同步更新 Model.md

</critical_notes>

<file_map>
MILP Model/CLAUDE.md - 本檔（天條）
optimfoundation-api-guide.md - 端到端 API 開發規範（建專案 → 資料/變數/模型層 → Program.cs 組裝 → 寫實驗 → 驗收 → API 速查/黑名單）；給人與 AI 照著寫的完整教學版
Model Design/CLAUDE.md - Phase 1：4 階段建模法、五段 Model.md（SET/PARAM/VAR/CONSTRAINT/OBJ）、程式轉譯 metadata、建模自驗 gate
Model Design/linearization-patterns.md - constraint 8 類 canonical form + 非線性→線性 recipe（abs/max/min/fixed-charge/either-or）+ Big-M 鐵律
Foundation Coding/CLAUDE.md - Phase 2：專案結構、generator vs 手寫、API 速查、禁止 API、解驗證協定
Foundation Tuning/CLAUDE.md - Phase 3：正確性 gate、三方向 tuning、Experiment API、stop conditions
evals/ - behavioral eval cases（改本 domain 規範後跑 `/harness-eval milp-model`）
../workflow skill/SKILL.md - milp-dev orchestrator skill（部署後觸發「幫我建模」等）

外部權威參考（$OPT，讀來參照不複製）：
$OPT/OptimFoundation/OptimFoundation/specs/developer-guide.md - 框架 13 章 API 手冊
$OPT/AI-Modeling/CPLEX_API_REFERENCE.md - 消費端 API 對照（含 soft constraint、Experiment）
$OPT/AI-Modeling/claudemdTemplate/ - 專案內各資料夾 CLAUDE.md 模板（單一來源）
$OPT/AI-Modeling/Projects/HospitalRostering_Generator - 預設（generator + OptModel）可運作範例
$OPT/AI-Modeling/Projects/HospitalRostering_Manual - 手寫後路可運作範例
$OPT/AI-Modeling/dlls/ - DLL 唯一來源（HintPath 一律指向這裡）
</file_map>

<paved_path>
## 標準流程（新題目）

1. **Model Design**：讀 `Model Design/CLAUDE.md` → 4 階段降維（去故事化+單位正規化 → 語義判別+Terminology 表 → SET/PARAM/VAR/CONSTRAINT/OBJ 抽取 → 建模自驗）產 `Model/<Project>_Model.md`（LaTeX，每條 constraint 標 pattern tag，手法見 `Model Design/linearization-patterns.md`）→ 歧義追問 → 等使用者確認
2. **Foundation Coding**：讀 `Foundation Coding/CLAUDE.md` → 在 `$OPT/AI-Modeling/Projects/<Project>/` 依五資料夾結構逐條轉譯 → `dotnet build` → fix loop（≤5 次）→ `dotnet run` → 解驗證協定（四步）
3. **Foundation Tuning**（使用者提出才做）：讀 `Foundation Tuning/CLAUDE.md` → 正確性 gate → 依觸發類型走 solver / IIS / structure 路線

整段流程由 `milp-dev` skill orchestrate（含 resume）；手動走也依同順序。
</paved_path>

<patterns>
## 命名對應（Model.md 符號 → 程式類別）

| Model.md 符號 | 程式類別 |
| --- | --- |
| `Produce_{GlassType}`（連續量） | `VariableX_Production` |
| `Assign_{Employee,Date}`（0/1） | `VariableB_Assign` |
| `Capacity_{GlassType}` constraint | `Constraint_Capacity` |
| `DemandQty` parameter | `Parameter_Demand`（值放 `QTY` 欄位） |

## Pool API 最小速查（詳見 Foundation Coding）

```csharp
// Model: sum_g Produce_g * UsageRate_g <= MachineCapacity
foreach (var g in dataload.GlassTypes)
{
    var rate = dataload.parameter_UsageRate.FirstOrDefault(p => p.GLASS_TYPE == g)?.QTY ?? 0.0;
    engine.AddLHS(rate, new VariableX_Production { GLASS_TYPE = g });
}
var cap = dataload.parameter_MachineCapacity.First().QTY;
engine.AddRHS(cap);
engine.CreateLessEqual("MachineCapacity");
```

✅ Good：係數先 LINQ 存變數再傳入；❌ Bad：LINQ 直接內嵌 `AddLHS(...)`、`engine.AddRHS(40.0)` 裸數字。
</patterns>

<common_tasks>
- 新題目 / 「幫我建模」 → milp-dev skill（或手動走 paved_path）
- 使用者要改約束 / 嫌太慢 / infeasible → 先讀 `Foundation Tuning/CLAUDE.md` 再動手
- API 簽名不確定 → 查 `$OPT/AI-Modeling/CPLEX_API_REFERENCE.md`，NEVER 憑記憶寫
- 改本 domain 規範 → 跑 `/harness-eval milp-model` 回歸
</common_tasks>

<hatch>
- `AI-Modeling/automated/` 的全自動 16-stage 量產路線（Claude Code 驅動 `Prompts/`，不需人在迴路）用它自己的 `automated/CLAUDE.md`，不套本 domain 互動 phase gate
- 純數學討論 / 教學（不落地成專案）→ 不必 phase gate，但命名與 LaTeX 慣例照用
- 使用者一句話明說「模型我確認過了，直接寫 code」→ 視同通過 gate，直接進 Coding
</hatch>

<fatal_implications>
- NEVER 模型未確認就產 `.cs`
- NEVER 移項 / 改號 / 翻轉比較方向 / 四捨五入數值
- NEVER Constraint / Objective 出現裸數字
- NEVER 呼叫不存在的 API（`GetVarSol`、`GetSetVarSol`、`CsvCtrl.SaveToCSV` 皆不存在，見 Foundation Coding 禁止清單）
- NEVER 改 OptimFoundation 框架本體（唯讀）—— 要擴充在專案端寫 helper；真要改框架 → Core + Cplex DLL 一起 rebuild 並更新 `$OPT/AI-Modeling/dlls/`
</fatal_implications>
