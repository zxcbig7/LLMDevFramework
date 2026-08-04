---
title: OptimFoundation API Guide 嚴格化 — 以「自由度歸零」為判準的重寫
status: draft
created: 2026-08-03
updated: 2026-08-03
modules: [docs, milp-model-domain]
---

# OptimFoundation API Guide 嚴格化改版

## Summary

把 `MILP Model/optimfoundation-api-guide.md` 從「原則式規則書」重寫成「自由度歸零的骨架規範」。唯一判準：**任何兩個 AI 拿同一份 `Model.md`，產出的專案必須逐檔近乎相同**。凡是「兩種寫法都合規但長得不一樣」的地方就是一個自由度，MUST 被一條規則關掉。本規格的主體因此不是改進項清單，而是一張**自由度清單**（Part A）：逐檔型列出每個決策點與它收斂到的唯一寫法。附帶修掉 Part B 的七個既有跨檔矛盾，並補上 Part C 兩個讓規則得以成立的上游依賴。

## Motivation / Why

改版起點是使用者連續七次追問「X 有規定清楚嗎」（Dataload、namespace、Program.cs、DLL 引用、Program 變數、題目中立性…），每問一次就補一條規則。**這個模式本身就是證據：規則是被動打補丁補出來的，不是系統性盤出來的**——追問會一直有效，代表洞永遠找不完。

改採可判定的判準後，問題重新定義為：guide 現在容許多少「合規但發散」的寫法？實測 golden template 與 guide 正文比對，發現的發散點不是零星的——連 `Sudoku_SHC279` 自己在同一個 `Program.cs` 裡都對同一類旋鈕用了兩種拼法（`randomSeed` 原生欄位 vs `.Emphasis` 抽象別名）。所以要修的是「規範的覆蓋方式」，不是「多加幾條規則」。

## Scope

### In Scope

- 依 Part A 自由度清單重寫 `optimfoundation-api-guide.md`（先落地 `optimfoundation-api-guide_new.md` 供審閱）
- 修掉 Part B 的七個既有矛盾
- 補 Part C 的上游依賴（`Model Design/CLAUDE.md` 兩個 metadata 欄位）—— 原本規劃排除，但 Part A 的 A4/A6 兩條規則需要 `Model.md` 提供的資訊；不補則規則懸空無法機械執行，因此強制納入
- 同步天條層 `MILP Model/CLAUDE.md`、`Foundation Coding/CLAUDE.md`

### Out of Scope

- `linearization-patterns.md`（Phase 1 數學手法，與本次結構決定論無關）
- 不新增 `evals/` case；完成後建議跑 `/harness-eval milp-model` 回歸，但由使用者決定時機
- 不回頭改 `$OPT` 底下任何既有 code（不改 `Sudoku_SHC279` 或既有 `Projects/*` 去配合新規則）

## 核心方法（Part A）：自由度清單

每一列 = 一個 AI 會自行發揮的決策點。「現況」欄標示 guide 目前的狀態：`無`=完全沒規定、`隱含`=只能從範例猜、`衝突`=兩處規定不一致、`有`=已規定清楚（列出僅為完整性）。

### A0 · 專案層

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A0.1 | `<Project>` 識別碼怎麼取 | 無 | 單一合法 C# identifier（NEVER 空白／連字號／數字開頭）；逐字相同用於五處：資料夾名、`.csproj` 檔名、`RootNamespace`、`AssemblyName`、每個 `.cs` 的 `namespace` |
| A0.2 | 專案根可以存在哪些檔 | 衝突 | 白名單：`<Project>.csproj`、`Program.cs`、`TuningHistory.md` + 八個資料夾。§1.1 現行結構圖沒列 `TuningHistory.md`，卻在 §8 要求它 → 與「NEVER 增減」直接衝突，MUST 補進結構圖 |
| A0.3 | DLL 引用機制 | 有（但將被 golden template 破壞） | `..\..\dlls\` HintPath + `<Analyzer>`；★ NEVER 抄 golden template 的 `ProjectReference` / `$(CplexDir)`（見 Part B-1） |
| A0.4 | 要不要加 `<LangVersion>` | 無 | NEVER 加。net8.0 預設即 C# 12，primary constructor 不需額外開關 |
| A0.5 | csproj 其餘欄位 | 有 | 照抄 §1.4 模板，只改 A0.1 的三處名字 |

### A1 · 所有 `.cs` 共通

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A1.1 | 每種檔型該寫哪些 `using` | 隱含 | 新增「檔型 → 固定 using block」對照表（Set/Parameter/Variable → `OptimFoundation.Modeling`；Constraint → `Core` + `Cplex`；Objective → `Cplex`；Dataload → `Core` + `Core.IO`；Solution → `Core` + `Core.IO` + `Cplex`；Program → `Core` + `Cplex`）。ImplicitUsings 已提供 `System` / `Linq` / `Collections.Generic`，NEVER 另外手寫這些 |
| A1.2 | `using` 排序 | 無 | 字母序 |
| A1.3 | namespace 形式 | 有 | block `namespace X { }`，單一 namespace |
| A1.4 | 存取修飾字 | 隱含 | 一律 `public sealed`（generator 型別 `public sealed partial`）；唯一例外 `Program` 為 `internal static` |
| A1.5 | `<summary>` 內容契約 | 隱含（只說「要有」） | 一句話語意 + 明確寫出對應 `Model.md` 的哪個符號或哪條式子；Constraint 則貼該條數學式 |
| A1.6 | 縮排／括號風格 | 無 | 4 空格、Allman 大括號 |
| A1.7 | 檔案編碼 | 無 | UTF-8 無 BOM |
| A1.8 | 欄位對齊 | 無 | 單一空格，NEVER 為對齊上下行補空白（框架全域鐵則） |

### A2 · `Set/Set_*.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A2.1 | 元素型別怎麼選 | 無 | 由 `Model.md` SET 段的成員範例機械判定：類別型→`string`、序號型→`int`、日期型→`DateTime`；NEVER 自行改型別 |
| A2.2 | 是否寫裸 `[OptSet]` | 有 | ALWAYS 顯式 `[OptSet<T>]` |

### A3 · `Parameter/Parameter_*.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A3.1 | `[OptDim<Set_X>("...")]` 的字串取什麼 | 無 | = Set 類別名去掉 `Set_` 前綴，逐字相同；唯一例外：同一顆 Set 在同一類別出現兩次（角色名如 `LotFrom`/`LotTo`），此時角色名 MUST 已在 `Model.md` 定義 |
| A3.2 | 何時用 `HasValue = false` | 隱含 | `Model.md` PARAM 該列只表示「這組合存在」而無數值時 → `HasValue = false`；有數值 → 預設 |
| A3.3 | attribute 宣告順序 | 有 | = key 組成順序 = `Model.md` 的 Dim 標示順序 |

### A4 · `Variable/Variable[B|X|I]_*.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A4.1 | 前綴選哪個 | 有 | 由 `Model.md` VAR 的型別欄機械對應 |
| A4.2 | ★ LB/UB 何時要建 constraint | 無（重大缺口） | 三分規則：`VariableB_` → 前綴已給 `[0,1]`，NEVER 另建；`LB=0 且 UB=INFTY` → 不必寫任何東西；**其餘任何界限一律建一支 `Constraint_*.cs`**。此規則需 `Model.md` VAR 表提供 LB/UB（已有）——但該表目前措辭為「決定 `VariableB_/I_/X_` + bound overload」，`bound overload` 已被廢止，見 Part C-2 |

### A5 · `Data/Dataload.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A5.1 | 專案名稱字串怎麼來 | 無 | `public const string ProjectName = "<Project>";` 置於類別最頂端；與 A0.1 識別碼逐字相同；`Program.cs` / `Solution` / `Logging` / `OptExperiment` 全部引用它，NEVER 重複字面字串（現行 §5 範例重複打 `"MyProject"` 三處以上） |
| A5.2 | 欄位可不可以被重新指派 | 無 | 一律 `readonly` —— §0.2 現行只說「視為唯讀但編譯器不保證攔截」，等於君子協定；`readonly` 升級成編譯期保證，且不影響現有寫法（`Load(...)` 是就地填值；`LoadParam` 的指派發生在建構子內，C# 允許） |
| A5.3 | 類別裡還能放什麼 | 無 | 成員封閉為五種：`ProjectName` 常數、欄位、預設建構子、`IDataSource` 建構子、（選用）import 建構子 + `Export()`。NEVER 其他 method / property |
| A5.4 | 欄位宣告順序 | 無 | = 建構子內 `Load` 敘述順序 = `Model.md` SET 段順序接 PARAM 段順序 |
| A5.5 | 欄位型別 | 隱含 | `Set_X` 用 `Set_X`；`Parameter_X` 用 `List<Parameter_X>`。NEVER 用 `IReadOnlyList<T>` / `T[]` / `IEnumerable<T>` 包一層 |
| A5.6 | 建構子內能寫什麼 | 有 | 兩種敘述白名單（現行 §2.4 已規定） |

### A6 · `Constraint/Constraint_*.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A6.1 | 依賴注入寫法 | 衝突 | C# 12 primary constructor，NEVER 手寫 `private readonly` 欄位 + 建構子指派（一次消滅「底線 vs 無底線」的分歧，見 Part B-2） |
| A6.2 | primary ctor 參數順序 | 無 | Set（依 `Model.md` 的 ∀ 順序）→ `List<Parameter_*>`（依式子出現順序）→ scalar |
| A6.3 | 參數命名 | 衝突 | Set 參數 = 語意核心的 lowerCamelCase 複數；Parameter 參數 = 語意核心 lowerCamelCase，NEVER 加 `By<Dim>` 後綴（現行 §4.2 範例用 `maxDaysByItem`，golden template 用 `blockCells`，定於不加後綴的簡式） |
| A6.4 | primary ctor 參數可否重新指派 | 無 | NEVER 在 `Build` 內重新指派（C# 12 捕獲參數可變，是語言的坑） |
| A6.5 | foreach 巢狀順序 | 無 | = `Model.md` 該條 constraint 的 ∀ 標示順序 |
| A6.6 | 迴圈變數命名 | 隱含 | = 該 Set 語意核心的單數 lowerCamelCase |
| A6.7 | ★ 係數查表用哪種 LINQ | 衝突（語意不同！） | 由 `Model.md` PARAM 的 total/sparse 標示決定：total → `.Single(p => ...)`（缺資料當場炸）；sparse → `FirstOrDefault(...)?.QTY ?? <Model.md 明示的 default>`。現行 guide 用 `?? 0.0`、golden template 的 Objective 用 `.Single()`，兩者**失效語意完全不同**（靜默變成 0 vs 立刻失敗）。此規則需 Part C-1 的上游欄位 |
| A6.8 | 查到的值存哪 | 有 | 先存 local 再傳入，NEVER 內嵌 LINQ 進 `AddLHS` |
| A6.9 | 敘述順序 | 隱含 | LHS 全部 → RHS 全部 → `CreateXxx`，NEVER 交錯 |
| A6.10 | constraint 名稱格式 | 有 | `$"{ConstraintName}@{i}@{j}"`；`DateTime` 用 `yyyy_MM_dd`；複合索引用底線 |

### A7 · `Objective/ObjectiveFunction.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A7.1 | 依賴注入寫法 | 衝突 | 同 A6.1–A6.4（primary constructor） |
| A7.2 | min / max | 有 | 由 `Model.md` OBJ 方向機械對應 |
| A7.3 | 沒有 OBJ 段怎麼辦 | 有 | 停止 Phase 2 退回 Phase 1 |

### A8 · `Solution/<Project>Solution.cs`（★ 目前最鬆的一塊）

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A8.1 | 類別成員有哪些 | 無 | 白名單：private 欄位、private 建構子、`static ReadAndValidate(OptEngine, Dataload)`、`private static ValidateRules(...)`、`public Print()`。額外 private helper 僅在被 `ValidateRules` 重複呼叫時允許 |
| A8.2 | `ValidateRules` 要驗到什麼程度 | 隱含 | `Model.md` 的**每一條** constraint 各對應至少一個檢查；不成立 `throw`，NEVER 只寫 log 繼續 |
| A8.3 | ★ 容差／門檻裸數字 | 無（與禁令衝突） | `1e-6` 容差與 `> 0.5` binary 門檻是**數值方法常數，不是模型係數**，MUST 具名為 `private const double` 並在 guide 明確 carve out 於「常數位禁裸數字」規則之外——現行 §6 範例直接內嵌裸數字，與 §4.4 天條表面衝突 |
| A8.4 | `Print()` 能做多花 | 無 | 固定契約：輸出目標值 + 非零決策變數；NEVER 自創 ASCII 視覺化／盤面繪製（golden template 的格線輸出屬題目專屬，NEVER 一般化） |
| A8.5 | 是否可收整包 `Dataload` | 有 | 唯一允許的地方 |

### A9 · `Program.cs`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A9.1 | 段落切法與註解樣式 | 衝突 | 四段純文字編號註解 `// 1. 材料` `// 2. 模型` `// 3. 環境` `// 4. 正式求解`（採 golden template 實際形式，取代現行 §5 的三段裝飾線版本） |
| A9.2 | 材料段宣告順序 | 隱含 | `data` → 各 scalar（`.Single().QTY`，變數名 = 去 `Parameter_` 前綴的 lowerCamelCase）→ `projectConfig` → `productionBaseline` |
| A9.3 | `.AddVariables` / `.AddConstraints` 排列順序 | 無 | = `Model.md` VAR / CONSTRAINT 段的宣告順序，一對一 |
| A9.4 | ★ solver 旋鈕用原生欄位還是抽象別名 | 衝突 | **一律 `CplexConfig` 原生 camelCase 欄位**（`mipEmphasis` / `randomSeed` / `epGap` / `timeLimit` / `workThreads` / `parallelMode` / `probe`），NEVER 用 `Emphasis` / `Seed` 等 `ITunableConfig` 抽象別名。實測 `CplexConfig.cs:171` 證實 `Emphasis` 只是 `mipEmphasis` 的 alias property，golden template 自己在同一檔混用兩種拼法。Why：只有原生欄位覆蓋全部旋鈕（`probe`／cuts 族沒有抽象別名），混用會讓同一份 config 出現兩種拼法、promotion diff 難比對。guide MUST 附一張 alias 對照表（`Emphasis`↔`mipEmphasis`、`Seed`↔`randomSeed`），因為 experiment 的 CSV/JSON 快照記的是抽象名，AI 寫回 baseline 時要能翻譯 |
| A9.5 | provenance 註解 | 無 | `productionBaseline` 上方固定四行註解模板（用途、來源 experiment、promotion/retain 決策、指向 `TuningHistory.md`） |
| A9.6 | experiment 骨架 | 無 | 固定形狀：warm-up cell（標 `-warmup-exclude`）→ N 個 seed × 各 variant，且 variant 順序跨 seed 輪替；experiment 命名 `<Project>-tuning-r<N>` |
| A9.7 | 三態 CLI 判斷順序 | 有 | 現行 §5.1 已規定 |

### A10 · `TuningHistory.md`

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A10.1 | 檔案格式 | 無模板 | 固定：標題 + 固定 intro 段 + 每輪一個 `## r<N> — <promotion/retain/rejected>` + 固定欄位表 |
| A10.2 | 表格欄位 | 兩處不一致 | 統一為 `Foundation Tuning/CLAUDE.md` 已列的欄位集合（日期／experiment／baseline Trial／candidate Trials／instances·seeds·彙總方法／eligibility／Status·objective·gap／config diff／決策／理由／production 驗證），並與 golden template 的實際表格對齊 |

### A11 · 跨檔一致性

| # | 自由度 | 現況 | 收斂規則 |
| --- | --- | --- | --- |
| A11.1 | ★ 題目中立性 | 無 | guide 正文（尤其新增的 tuning 骨架、`TuningHistory.md` 模板、provenance 註解）一律用中性 placeholder（`<Project>` / `MyProject` / `Item` / `Date` / `Employee`），**NEVER 出現 sudoku／數獨／宮格／格子／rostering／班表等任何特定題型字眼**。從 golden template 抽取的是結構 pattern，NEVER 是字面內容（例如它的 `PuzzleName` 常數 → 規範寫成 `ProjectName`）。允許唯一例外：規則旁一句話註明「此骨架已於既有 template 驗證合規」作為來源佐證，且不得因此讓範例使用該題目詞彙 —— Why：混入單一題目字眼會讓 AI 誤判骨架只適用該題型，或把題目專屬命名當成規範要求的 identifier |
| A11.2 | `Model.md` ↔ code 對應 | 隱含 | guide 補一張總表：`Model.md` 每個元素 → 該產生哪個檔 → 檔內哪些欄位由 `Model.md` 哪一欄決定 |

## Part B：既有矛盾清單（MUST 一併修掉）

| # | 矛盾 | 證據 | 處置 |
| --- | --- | --- | --- |
| B1 | golden template 的 `.csproj` 用 `ProjectReference` 指向框架 src + `$(CplexDir)` 抓本機 CPLEX Studio 安裝路徑，正是 guide §1.4 對 `Projects/*` 明文禁止的兩件事 | `Sudoku_SHC279.csproj:19-23,27-32` | 升格 golden template 的同時，用同等顯眼篇幅劃出範圍：可模仿 C# 程式結構，NEVER 模仿其 `.csproj` 引用機制。否則新專案在 `Projects/` 底下直接建置失敗 |
| B2 | guide 範例私有欄位不加底線、golden template 全加底線 | §4.2/§4.3 vs `Constraint_BlockDigit.cs:9-12` | 改 primary constructor，兩者一起消滅 |
| B3 | guide 用三段裝飾線註解、golden template 用四段純文字 | §5 vs `Program.cs:22,44,79,130` | 採四段純文字 |
| B4 | 同一旋鈕兩種拼法：`mipEmphasis`（Foundation Tuning 症狀表）vs `Emphasis`（guide §5/§8、golden template） | `CplexConfig.cs:171` 證實是 alias | 一律原生 camelCase，附 alias 對照表（A9.4） |
| B5 | 係數查表 `.Single()` vs `FirstOrDefault ?? 0.0` 並存，失效語意完全不同 | §4.2 vs `ObjectiveFunction.cs:31` | 由 `Model.md` total/sparse 標示決定（A6.7 + Part C-1） |
| B6 | §1.1 資料夾結構未列 `TuningHistory.md`，但 §8 要求建立它，與「NEVER 增減」衝突 | §1.1 vs §8.1 第 6 點 | 補進結構圖白名單 |
| B7 | §6 `ValidateRules` 範例內嵌裸 `1e-6` / `> 0.5`，與 §4.4「常數位禁裸數字」天條表面衝突 | §6 範例 | 具名 `const` + 明確 carve out 數值方法常數 |

## Part C：上游依賴（`Model Design/CLAUDE.md` 需補）

Part A 有兩條規則需要 `Model.md` 提供目前不存在的資訊，不補則規則無法機械執行：

| # | 缺口 | 影響的規則 | 處置 |
| --- | --- | --- | --- |
| C1 | PARAM metadata 表沒有「total / sparse + default」欄位 | A6.7 無法判定該用 `.Single()` 還是 `?? default` | PARAM 表加一欄；`common_tasks` 的預設慣例「參數未定義的組合→填不影響模型的預設值」MUST 改成「MUST 在 PARAM 表明示 total 或 sparse+default」 |
| C2 | VAR metadata 表措辭仍寫「決定 `VariableB_/I_/X_` + **bound overload**」，但 bound overload 已在天條層廢止（界限一律寫成 constraint） | A4.2 讀到過時指示 | 改成「LB/UB → 依 A4.2 三分規則決定是否產生 `Constraint_*`」 |

## Acceptance Criteria

- [ ] Part A 的 **每一列**都能在新 guide 找到對應的明文規則（逐列對照，不是抽樣）
- [ ] Part B 七個矛盾全部修掉，且修完後三份天條文件與 guide 互不衝突
- [ ] Part C 兩個上游欄位補入 `Model Design/CLAUDE.md`
- [ ] 新 guide 全文搜尋不到 `sudoku`／`數獨`／`宮`／`rostering`／`班表`等題型字眼（A11.1）
- [ ] guide 含「檔型 → 固定 using block」對照表、`Model.md` ↔ code 對應總表、solver 旋鈕 alias 對照表三張新表
- [ ] `Foundation Coding/CLAUDE.md` 改完仍 ≤200 行
- [ ] 通過 `prompt-principles/self-check.md` 12 點
- [ ] 反向驗收：拿新 guide 對照 golden template 逐檔核對，凡 golden template 與新規則不符之處（已知至少 `.csproj`、`.Emphasis` 拼法、`PuzzleName` 命名、`Print()` 盤面繪製四處），guide MUST 已明文標示為「不可模仿」

## Edge Cases & Error Handling

- generator 型別（Set/Parameter/Variable）NEVER 套用 primary constructor 規則——它們是 `sealed partial class X { }` + attribute，無手寫建構子
- Solution 類別用「靜態工廠 + private 建構子」，NEVER 套 primary constructor
- 小規模題目（秒解）仍 MUST 建 `TuningHistory.md`，但允許記錄成「單一 baseline retain，未做 seed 掃描」的簡化版；guide 需註明何時可簡化 A9.6 骨架，避免小題被迫套 3-seed 輪替造成過度工程
- `Model.md` 缺 Part C 所需欄位（舊專案）→ 停下回 Phase 1 補，NEVER 自行猜 total/sparse

## Non-Functional Requirements

- **文件長度**：`optimfoundation-api-guide.md` 是教學擴充版（非 domain CLAUDE.md），可維持長篇；`MILP Model/CLAUDE.md`、`Foundation Coding/CLAUDE.md` MUST ≤200 行
- **風格**：對照 `prompt-principles/CLAUDE.md`——NEVER/ALWAYS/Why 三段式、good/bad 對照、explicit 可驗證用語
- **可驗證性**：每條規則 MUST 寫成「看 code 就能判斷有沒有違反」的形式，NEVER 用「合理」「適當」這類無法判定的字眼

## Implementation Plan

### Stub 階段（文件交付物：先出骨架不寫正文）

- [ ] 產出新 guide 的章節大綱 + 每節掛哪幾條 Part A 規則（對照表形式）
- [ ] 列出三份天條文件要改的具體段落
- [ ] 給使用者確認結構後才展開正文

### 逐步實作

- [ ] 展開 `optimfoundation-api-guide_new.md` 正文
- [ ] 補 `Model Design/CLAUDE.md`（Part C）
- [ ] 同步 `MILP Model/CLAUDE.md`、`Foundation Coding/CLAUDE.md`
- [ ] 跑 self-check + Part A 逐列對照 + 反向驗收（golden template 差異點）
- [ ] 建議使用者另找時間跑 `/harness-eval milp-model`

## References

- 本規格由逐檔核對 golden template 全部原始碼（`Program.cs`、`Dataload.cs`、`Constraint_BlockDigit.cs`、`Sudoku_SHC279Solution.cs`、`ObjectiveFunction.cs`、`Set_Block.cs`、`Parameter_BlockCell.cs`、`VariableB_CellDigit.cs`、`.csproj`、`TuningHistory.md`）+ `optimfoundation-api-guide.md` 全文 1026 行 + `MILP Model/CLAUDE.md` + `Foundation Coding/CLAUDE.md` + `Model Design/CLAUDE.md` + `Foundation Tuning/CLAUDE.md` + 框架原始碼 `CplexConfig.cs` 比對得出
- 相關既有規格：`specs/2026-07-04-milp-model-domain.md`、`specs/2026-08-01-optimfoundation-dual-config.md`、`specs/2026-08-01-optimfoundation-runner-symmetry.md`
