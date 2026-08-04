# Model.md → Code 驗收 Checklist（Phase 2 交付後，人工核對用）

<system_context>
Phase 2（Foundation Coding）AI 宣稱完成後，你（人）用這份表逐項核對，抓「AI 自己說過了、但實際沒過」的情況。
天條全文見 `../CLAUDE.md`；本表只是把天條轉成可勾選的檢查點，NEVER 只看 AI 的文字回報就結案——要求打開檔案實際看。
</system_context>

## A. 結構 / 命名

- [ ] 八資料夾都在（Set/Parameter/Variable/Objective/Constraint/Solution/Data/Model），沒有多資料夾
- [ ] 一型別一檔，檔名 = 類別名；`Model/` 內只有 `<Project>_Model.md`
- [ ] namespace 是專案名單一層（`namespace X { }`），沒有子 namespace
- [ ] 每個型別 `sealed`，`<summary>` 標對應 Model.md 符號
- [ ] Variable 前綴對型別：`VariableB_`=binary、`VariableX_`=continuous、`VariableI_`=integer

## B. 逐條對照 Model.md（最花時間但最重要）

- [ ] 每個 `Constraint_*` 都能在 Model.md 找到對應式子，數量一致（沒漏、沒多造）
- [ ] 比較方向沒被翻轉：`<=`→`CreateLessEqual`、`>=`→`CreateGreatEqual`、`=`→`CreateEqual`
- [ ] LHS/RHS 沒被移項/化簡——code 裡 `AddLHS`/`AddRHS` 的項目排列跟 Model.md 式子一模一樣
- [ ] `ObjectiveFunction` 是 OBJ 段逐項轉譯，沒有偷加、漏掉、或係數對調的項

## C. 禁止 Hardcode

- [ ] 常數位（單參數形式，如 `AddRHS(40)`）沒有裸數字，全部來自 `Parameter.QTY`
- [ ] 結構常數/迴圈邊界（例如「3×3 宮」的 3、時間窗長度 7）不是寫死數字，是 `Set` 或 `Parameter`
- [ ] 變數界限沒藏進 `BuildCVs(lb, ub, ...)` 這類已禁用參數，界限是獨立一條 constraint

## D. API 黑名單（出現任一直接打回）

- [ ] 沒有 `GetVarSol` / `GetSetVarSol` / `SaveToCSV` / `BuildBVs|CVs|IVs` / `CreateXxx(rhs, name)` / `LoadCsv` / `LoadInline` / Parameter 位置式 ctor
- [ ] `Dataload(IDataSource)` 內只有 `Load(...)` / `LoadParam<T>(...)`，沒有 if / 迴圈 / `Random` / 日期運算 / 補值

## E. import 模式（只有寫了 `Dataload(string rawFile)` 才檢查）

- [ ] `Export()` 寫出的每個名稱，都能在 `Dataload(IDataSource)` 找到同名的 `Load` / `LoadParam`，且數量相等 —— 少一顆時 import 仍 exit 0 看似成功，要到下次求解才炸
- [ ] 生成 / 攤平邏輯全在 `Data/Dataload.cs` 的 import ctor 內，沒有另開 `DataGenerator.cs` 或 `Data/import/`
- [ ] rawFile 實際存在於 `Data/raw/`；生成假資料的情境也有（規模、seed、分佈寫成檔案），否則這批資料不可重現
- [ ] `Export()` 末尾有 log 產出清單，import 跑完當場看得出少了誰
- [ ] 產出的 CSV 若要當正式 input，已從 `bin/.../Data/` 搬回專案 `Data/` —— 留在 bin 會被 `dotnet clean` 清掉
- [ ] import 與求解是分兩次命令跑的，沒有做成「import 後自動 solve」的複合模式

## F. Program.cs 組裝

- [ ] 只有 `Program.cs` 知道 `Dataload`；Constraint/Objective 建構子沒有整包收 `Dataload`
- [ ] `OptEngine` 只從 `Build(engine)` 傳入，沒有進建構子
- [ ] 材料 → 模型 → 環境三段式，沒有用 helper/local function 包裝組裝順序

## G. 解驗證（`dotnet run` 之後）

- [ ] 沒有因為 `Status == Optimal` 就宣稱完成
- [ ] 解有代回每條 constraint 驗可行（`ValidateRules`），不是只看目標值
- [ ] 目標值/關鍵變數的單位、量級跟題目描述對得上
- [ ] 有 LP bound sanity 或手算小例對照
- [ ] 若 `Infeasible`：有去讀 `IISs/*.ilp`，不是猜哪條約束錯

## 用法

每次 Phase 2 交付（或 AI 說「build 過了 / 解出來了」）時，打開 Model.md + 生成的 `.cs` 檔，逐項打勾；A~F 靠肉眼比對程式碼，G 靠實際跑出來的 log/輸出檔。任何一項打不了勾 → 退回 AI 重做該項，NEVER 因為「大概率是對的」跳過。
