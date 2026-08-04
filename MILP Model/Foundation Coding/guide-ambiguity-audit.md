# optimfoundation-api-guide 模糊點審查清單

> **審查標的**：`optimfoundation-api-guide_new copy.md`（2210 行）
> **審查標準**：笨模型照著寫也要能穩定輸出、誤差不大的 code
> **審查角度**：找出「還有辦法自己想、自己發揮」的漏洞
> **範圍**：只收**模型開發者**（寫專案 code 的人／AI）會遇到的問題。**OptimFoundation 框架視為固定不可改**——所有修法一律是「文件端補規則、補範例、補檢查步驟」，NEVER 提議改框架行為。框架的靜默路徑不當成 bug 報，而是當成「開發者必須知道的症狀 + 自保步驟」寫進規範。
> **產出**：54 項，收斂成 8 個決策包 + 4 個矛盾（矛盾 1、3、4 已裁決）
> **日期**：2026-08-04

---

## 一、先做這件事（5 分鐘）

**#28 文件裡有一個事實錯誤，錯的範例比沒有範例更糟。**

§1.5 的 `RegisterAll` 產出範例寫成 `RegisterSet("Set_Item", set_Item)`，generator 實際 emit 的是去掉前綴的 `RegisterSet("Item", set_Item)`（`AutoSetsGenerator.cs:404`）。

這不是細節：`RegisterParam` 的 `indexSets` 用的是**維度名**（`new[] { "Item", "Date" }`），驗證時拿維度名去查 sets 字典，所以註冊名必須去前綴才對得上。範例寫錯會讓人以為註冊名與檔名是同一個字串。

---

## 二、四個矛盾，補丁前要先裁決

### 矛盾 1 · 附錄 A 的 code 骨架會反噬（#13 ↔ #33）—— 已裁決 2026-08-04

在 Phase 2 文件放 pattern 的 code 骨架，等於暗示「遇到非線性可以自己套 pattern」，正好打開 #33 要堵的洞。

**裁決（Vic）**：非線性在 Phase 1 就該把關完畢，寫 code 不管這件事。據此：

| 附錄 A | 屬於誰 | 處置 |
| --- | --- | --- |
| **A.1** 8 類 canonical form | 兩邊都用 | **留** —— Phase 2 拿它「認形狀 → 選轉譯骨架」 |
| **A.2** 非線性 → 線性 recipe | **Phase 1 專屬** | **移除**（或降級為「你會在 Model.md 看到這些形狀」的對照，NEVER 寫成可操作的配方） |
| **A.3** Big-M 鐵律 | 兩邊都用 | **留** —— Phase 2 要驗 M 是不是具名 parameter |

理由：Model.md 若已線性化完成，Phase 2 看到的只會是**普通線性式**（`w ≤ z₁`、`w ≥ z₁ + z₂ − 1` 已是三條獨立 constraint，各有 pattern tag），根本不需要知道它們原本是一個乘積。給 recipe 只會讓模型在 Model.md 沒拆乾淨時「順手幫忙拆」——那就是在 code 層改模型。

**Phase 2 需要的只有一句話，不是一套 recipe**：看到變數 × 變數、除法、`abs`、`max` / `min` → 這條 Model.md 沒完成 → 停止，退回 Phase 1。

**連帶解掉**：

- **#14** 輔助變數（`t`、`w`、`u`）必然已在 Model.md 的 VAR 段具名宣告，Phase 2 照 §3 規則建即可，不存在「誰負責加」。附錄 A 的單字母符號隨 A.2 一起移除。
- **#49** MTZ 的 `u ∈ [1, n−1]` 兩條界限式子，Model.md 就該寫出來，Phase 2 逐條轉譯。
- **#13** 降級：只需為 A.1 的 8 類提供轉譯骨架，不必為 A.2 的 7 種 recipe 提供。

### 矛盾 2 · Big-M 的來源有兩條路（#15）

- 附錄 A.3：「MUST 定義成 `Parameter_*` 從 CSV 讀入」
- §2.4:748：「由資料推導的比值（Big-M 之類）用 `Numeric.SafeRatio(...)`」

一個說是資料、一個說是算出來的，沒說何時走哪條；而且 SafeRatio 那條要寫在 `Dataload` 裡，但白名單只准兩種句子，`BigM` 到底是 property 還是別處，沒有範例。

**裁決方向**：訂單一來源 + 優先序（先找既有 parameter 當 M → 找不到才由資料推導 → 推導不出才新建），新建時 `<summary>` MUST 寫明從哪個量推來。

### 矛盾 3 · Set 順序語意二選一（#39）—— 已裁決 2026-08-04

- **A 案**：Set 一律視為無序，順序關係做成 `Parameter_NextDate` —— 正確但每個時間題都要多一顆 parameter，且要 Phase 1 產
- **B 案**：需要順序時 MUST `.OrderBy(...)` —— 較輕，但對 string set（`"P1","P10","P2"`）會排出錯誤順序

**裁決（Vic）**：採 A 案。

否決 B 的理由（依份量排序）：

1. `.OrderBy(...)` 本身就是 Phase 2 的自行詮釋——排序鍵、升冪降冪、`DateTime` 有無時間部分全是模型語意，卻在 code 層拍板，直接違反天條「Coding 是 Model.md 的純機械轉譯」。
2. `OrderBy` 只給「決定性」，不給「相鄰語意」：排完序仍不知道「下一期」是*序列的下一個*還是*日曆 +1 天*，#4 / #42 原封不動留著。
3. 「只有 `DateTime` / `int` / `long` 能 `OrderBy`，string 走 A」= 同一題兩種寫法，正是本審查要消滅的東西。

**據此的條文**：

| 位置 | 內容 |
| --- | --- |
| Phase 2 天條 | NEVER 依賴 Set 的元素順序：禁 `set[i]`、`set[i+1]`、`.OrderBy`、以 `.First()` / `.Last()` 當期初 / 期末。Set 只准 `foreach` 與 `.Count` |
| Phase 2 天條 | 順序 / 相鄰 / 期初期末 / wrap-around 語意 MUST 來自 Model.md 宣告的 parameter：`Parameter_AdjacentDate{Date, NextDate}`（key-only）、`Parameter_InitialDate{Date}`、`Parameter_FinalDate{Date}` |
| Phase 2 天條 | 消費方式 MUST 迭代 parameter 資料列，NEVER 迭代 `Set × Set` |
| §2.0:339 | 「保序」改寫為：讀入順序 = CSV 行序，但 NEVER 依賴它——換一批未排序的 CSV 不得改變模型 |
| §0.0 進場檢查 | 加一條（同時落地 **#43**）：Model.md 出現相鄰期 / 前一期 / 累積 / 期初期末 → MUST 有對應順序 parameter，否則退回 Phase 1 |

**連帶解掉**：#4、#42 三選一消失（只剩顯式 parameter 一條路）、#43 有了具體著陸點。

**成本評估**：順序關係就是稀疏的二元關係，與 `Parameter_Arc{From,To}` 同型，與包 1 的稀疏總則共用同一個消費 pattern——等於一條總則同時關掉包 2 的一半。無順序語意的題目（指派、網路流）一顆都不用加，成本只落在時間題，而那正是靜默變成另一題的風險最高處。

**前置相依**：本規則要能執行，包 1 的 **#45**（key-only parameter 怎麼在限制式裡消費）MUST 先補範例，否則規則寫了沒有可抄的形狀。**Phase 1 文件連動**：`Model Design/CLAUDE.md` 要加「時間 / 順序題 MUST 在 PARAM 段宣告順序 parameter」。

### 矛盾 4 · tuning 規則要求了結構做不到的事（#21 ↔ #22/#23）—— 已裁決 2026-08-04

§8.2 要求「3–5 seeds 與 hold-out instances」，但多 instance 需要載入多份 data，直接撞上 §8.1「所有 cell 共用一份 `OptData.Load`」與 §5「材料段只 Load 一次」。

**裁決（框架固定 → 不改結構）**：把做不到的要求從規則降級或移除，只留開發者實際做得到的：

| §8.2 現有要求 | 處置 |
| --- | --- |
| 3–5 seeds | **留** —— `Clone()` 改 `randomSeed` 就做得到 |
| hold-out instances | **移除** —— 單一 `OptData.Load` 結構下做不到 |
| shifted geometric mean / PAR10 | **移除或降級為註記** —— 沒有實作位置（八資料夾裡沒有它的家），留著只會被跳過 |
| 第一個 solve 當 warm-up 排除 | **留** —— 執行順序安排得到 |
| 跨 seed 輪替 / 隨機化執行順序 | **留** |
| 「改善須大於 baseline variability」 | **改寫成可執行版** —— 例如「同一 config 跑 3 個 seed，改善幅度須大於這 3 次自身的全距」 |

原則：**規則只寫開發者在現有結構下做得到的事**。做不到卻寫進 MUST，等於訓練人跳過規則。

---

## 三、54 項收斂成 8 包

| 包 | 含哪些 | 要不要連動改 Phase 1 文件 |
| --- | --- | --- |
| **0. 立即修錯誤** | #28 | 否 |
| **1. 稀疏立場**（最大） | #1 #2 #44 #45 #47 #48 | **要** —— Model.md 的 CONSTRAINT 要標 `∀` 的 domain 是 Set 全集還是某 parameter 的 support |
| **2. 時間與順序**（矛盾 3 已裁決為 A 案，只剩補範例） | #39 #4 #42 #43 已定案，待補 key-only 順序 parameter 的消費範例 | **要** —— PARAM 段宣告順序 parameter + 進場檢查加「時間邊界已明確」 |
| **3. 線性化責任歸屬**（矛盾 1 已裁決，大幅縮小） | #33（進場檢查加一條 + 移除 A.2）；#13 降級為只補 A.1 骨架；#14 #49 已解 | **要** —— 輔助變數 MUST 在 Model.md 的 VAR 段宣告 |
| **4. 無訊號坑補完**（本包已成最大風險群） | #26 #27 #29 #50 #52 #53 #54 | 否 |
| **5. 數值與轉型** | #5 #6 #15 #34 #40 #46 | 否（#15 是矛盾，先裁決） |
| **6. 輸出與驗收契約** | #7 #8 #9 #10 #17 #19 #20 #36 | 否 |
| **7. tuning 可執行性**（矛盾 4 已裁決為降級） | #21 #22 #23 降級／移除；#24 #25 仍要補 | 否 |
| **8. 框架能力邊界的用語** | #51 | 否 |

**包 1、2、3 的根不在 Phase 2**，硬在這份文件補只會補出更多矛盾。

### 建議處理順序

1. 包 0（修錯誤）
2. 裁決矛盾 2（僅剩這一個未決）
3. 包 3 —— 移除 A.2、進場檢查加「非線性一律退回 Phase 1」，工作量小且已定案
4. 包 1（稀疏立場）—— 訂一條總則，比逐條補範例省力；這是現在最大的一包，且 **#45 是包 2 的前置**
5. 包 2 —— 依矛盾 3 的 A 案條文落檔（含 §2.0 保序改寫、進場檢查、Phase 1 連動）
6. 包 4、5、6 —— 機械補完，可派 subagent 批次做
7. 包 7 —— 依裁決結果補齊或降級

---

## 四、54 項明細

### 稀疏立場（包 1）

**#1 係數查表句型沒定義語意** — §4.2:1047、§4.4:1119、§6:1401
`FirstOrDefault(p => p.Item == item)?.QTY ?? 0.0` 在三個範例出現，沒有一條規則說它是唯一句型。`First()` / `Single()` / `Sum()` / `?? 0.0` 四種在「查不到」時行為完全不同。更關鍵：稀疏參數查不到時 `0` 對某些式子是「需求為零」（正確），對另一些是「約束消失」（模型變鬆）。
→ 分三種情境定死：**必存在**用 `.Single(...)`；**缺 = 0 有意義**用 `?? 0.0` 且該行 MUST 註明；**缺 = 不該建這條式子**改成迭代資料列。

**#2 限制式展開範圍沒規定：迭代 Set 還是迭代資料列** — §4.2:1042 vs §2.3:675
兩種寫法都在文件裡出現過，沒有選擇規則。這直接決定建出幾條限制式。
→ MUST 依 Model.md 該條 constraint 標的 `∀` 範圍決定，沒標就退回 Phase 1。

**#44 覆蓋型約束遇稀疏 RHS 建不建**
`∀date,shift: Σ_e Assign ≥ Demand_{d,s}`，Demand 稀疏。迭代 Set 會建出 `Σ ≥ 0` 的無效約束（模型變大不變質）；迭代資料列則少建。兩種都「對」，但模型規模差很多，tuning 階段會誤判成 solver 問題。

**#45 純 key 參數（`HasValue = false`）怎麼用，全篇沒有範例**
教了宣告、CSV 形狀、表頭強制、載入句型，就是沒教怎麼在限制式裡消費。`Parameter_Arc{From,To}` 這種「哪些組合存在」的資料，寫成 `Travel ≤ ArcExists` 時**沒有 `QTY` 可放進 `AddRHS`**；改成迭代資料列則變數已被 `BuildVars` 全展開。這是 MILP 最常見的資料型態之一，覆蓋為零。

**#47 同一條 constraint 內混合「迭代 Set + 迭代資料列」沒有範例**
流量守恆是 `∀node`（Set）內層 `Σ_{(j,i) ∈ Arc}`（資料列）。文件兩個範例各只示範一種純形式，而稀疏網路題幾乎每條式子都是混合的。

**#48 目標式的展開範圍也沒規範**
#2 只講 constraint。`Σ Distance_{f,t}·Travel_{f,t}` 同樣面臨迭代 `Set × Set`（撞到不存在的弧）還是迭代資料列的選擇。§4.3 只說「逐項機械轉譯」。

### 時間與順序（包 2）

**#39 Set「保序」的語意沒定義，模型依賴順序時完全無保護** ★ 最高風險之一
§2.0:339 說 Set CSV「保序」，但只保證讀入順序 = CSV 行序，不保證排序過。`dates[i+1]` 取「第 d+1 天」等於依賴 CSV 行序：**換一批沒排序的 CSV，模型悄悄變成另一題**，而 index 參照、全格覆蓋、`ValidateRules` 全都抓不到（每個值都合法，只是順序不同）。
已驗證 `SetBase<T> : IReadOnlyList<T>`（`SetBase.cs:41`），`Count` 與 `this[int]` 都是公開的——索引存取是現成、最自然的寫法，笨模型不會繞路。
→ **已裁決（矛盾 3，A 案）**：Set 一律無序，順序語意一律走顯式 parameter。

**#4 / #42 相鄰期沒有 canonical pattern**
三種寫法都可行、失敗模式各不同：索引相鄰（撞 #39）、`date.AddDays(1)` + `Contains`（日期不連續時靜默少建）、顯式 `Parameter_NextDate`（正確但要 Phase 1 配合）。文件零範例。
→ **已由矛盾 3 收斂為單一路徑**：只留顯式 `Parameter_AdjacentDate`，另兩種寫法列入禁令；仍需補一個消費範例。

**#43 進場檢查缺「時間邊界已明確」**
最後 N 天不成完整窗、期初期末特例、要不要 wrap-around —— §0.0 的九條進場檢查沒有一條在問。Model.md 沒交代時 Phase 2 只能自己選，選哪個都改變限制式條數。
→ **已由矛盾 3 給出著陸點**：進場檢查加「順序 / 邊界 parameter 已宣告」，未宣告即退回 Phase 1。

### 線性化責任歸屬（包 3）

**#13 附錄 A 15 種 formulation 全部沒有 code 骨架** — §附錄A:2116-2143
文件要求每條 constraint 標 pattern tag、「照 template 填空、不 freehand」，但填空的目標形狀不存在——8 類 canonical form + 7 種 recipe 全是 LaTeX，沒有一行 Pool API。`w ≤ z₁`、`w ≥ z₁ + z₂ − 1` 這種三四條一組的式子，迴圈怎麼包、命名怎麼取、幾個檔，全靠自己想。
→ 補骨架後連帶鎖住 #2、#4。但務必先看矛盾 1。

**#14 附錄 A 自己違反命名天條，輔助變數沒有歸屬**
A.2 通篇用 `t`、`w`、`y`、`z₁` 單字母，天條是「NEVER 單一字母符號」。更麻煩的是輔助變數是誰的責任沒定：附錄 A 說形狀對不上要退回 Phase 1，但 §0.0 進場檢查沒有「輔助變數已在 VAR 段宣告」這條，於是笨模型會在 Phase 2 自己補一顆 `VariableX_T`，等於在 code 層改模型。

**#33 Model.md 出現「變數 × 變數」時誰負責線性化沒寫死**
`Produce ≤ Capacity · Open` 這種乘積形式進來時，附錄 A 看起來像叫你在 code 拆，天條說要退回 Phase 1，§0.0 進場檢查又沒有「不得出現變數 × 變數」這條。三處互不指涉。
→ 進場檢查加一條「非線性項（變數乘積、除法、abs、max/min）一律視為 Model.md 未完成，退回 Phase 1」。

**#49 輔助變數的界限式子算不算「Model.md 的一條」**
MTZ 的 `u_i ∈ [1, n−1]` 依 §3 要拆成兩條 constraint。誰宣告 `u`？界限要不要出現在 Model.md 的 CONSTRAINT 段、各自一個檔？Model.md 用值域符號表達時，Phase 2 拆成兩條算不算自行詮釋？沒有授權條款。

### 無訊號坑（包 4）

**#50 目標式建不出來時只 Warn，不報錯** ★ 第四個無訊號坑
`EngineBase.cs:772`：`_lhsTerms.Count == 0` → `Logging.Warn(... result=skipped)` → `return`。
`ObjectiveFunction.Build()` 一項都沒加（迭代空 Set、係數全 `?? 0.0` 卻無變數項）時，目標式**根本不建立**，模型退化成純可行性問題，`Status` 仍是 `Optimal`、目標值 0。
→ 進 §7 驗收：MUST 確認 log 有 `[目標式建構完成] … terms=N result=success` 且 N > 0，看到 `result=skipped` 就是沒建起來。

**#52 限制式同名 → 靜默跳過，而且回傳 `true`** ★ 最容易踩、後果最大
`EngineBase.cs:659-678`：`if (!_verifyConstraints.Contains(name)) { 建立 } else { LogDuplicateConstraint(name); }`，最後**不論有沒有建立都 `return true`**。
迴圈內命名忘了帶索引（`$"{ConstraintName}"` 而非 `$"{ConstraintName}@{item}"`）→ 只有第 1 圈建立，其餘全部被跳過，只留 `[CONSTRAINT_DUPLICATE] … result=kept_existing` 的 Warn。模型少掉幾乎整條約束族，solver 回 `Optimal`，解「好得不合理」。
§4.2:1059 有規定命名 MUST 帶索引，但完全沒說忘了帶的後果——規則有寫、後果沒寫、失敗靜默。

**#53 空 pool → 限制式不建立，只有 Warn**
`EngineBase.cs:526` `CheckHasPool`：`_lhsTerms` 與 `_rhsTerms` 都空 → `Logging.Warn([CONSTRAINT_EMPTY] … result=skipped)` → 不建立。
迭代到空 Set、或係數查表全部落空導致一項都沒加時，這條限制式直接不存在。`AddLHS` 也不會救——它只在 `varSpec == null` 時回 `false` 並 Warn 跳過該項（`EngineBase.cs:556`），變數查不到才 throw。
附帶：`AddLHS` / `AddRHS` / `CreateXxx` 都有 `bool` 回傳值，文件從未提過它們存在、也沒說 `false` 代表什麼，所有範例一律忽略。

**#54 §7 驗收沒有「檢查 log」這一步**（#50 #52 #53 的共同著陸點）
框架建模層有一套 `[…] result=skipped` 的 Warn 慣例（`CONSTRAINT_EMPTY` / `CONSTRAINT_DUPLICATE` / `VARIABLE_NULL` / 目標式 `terms=0`），但 §7 四步驗收查的是 Status、代回、單位、bound——**全都在「模型已經建對」的前提下才有意義**。模型少建了限制式或目標式，四步都不會發現。
→ §7 加**第 0 步**（純文件端、不需改框架）：build 完、solve 前掃 log 確認沒有 `result=skipped` / `CONSTRAINT_DUPLICATE` / `CONSTRAINT_EMPTY` / `VARIABLE_NULL`，並核對限制式建構條數與 Model.md 展開後的預期數量一致。這是唯一能抓到這三個坑的地方。

**#29 「`Dataload` 欄位 MUST public」這條規則不存在**
generator 只掃 public 且非 static 的 field / property（`AutoSetsGenerator.cs:394-398`）。寫成 `private` / `internal` → **靜默跳過，連 `OPTF006` 都不會報**（診斷在取得 memberType 之後才判斷，private 成員更早就 `continue`）。這是第三個無訊號坑，文件連提都沒提。
附帶：`IReadOnlyList<Parameter_X>` 也被接受，文件只寫 `List<>`。

**#27 §1.6 說「三種寫法不可混用」但沒說後果**
實際是靜默忽略：generator 先看有沒有 `[OptDim]`，有就整個走 `[OptDim]` 路徑，字串式參數被無聲丟掉。不報錯、不警告。

**#26 `Generated/` 實際是多層路徑，文件只寫一層**
除錯用法叫人「去 `Generated/` 找對應的 `.g.cs`」，實際會產在 `Generated/<analyzer 組件名>/<generator 全名>/<檔名>.g.cs`。沒說要往下鑽，照做的人會以為沒產出。

### 數值與轉型（包 5）

**#5 `ValidateRules` 的容差 `1e-6` 是魔術數字** — §6:1405
沒說哪來、要不要跟 `epRHS` 對齊。填 `1e-9` → 浮點誤差誤判違反；填 `0.01` → 真違反被吞。
→ 定成具名常數並對齊 CPLEX `epRHS` 預設。

**#6 binary 判定閾值 `> 0.5` 沒定義** — §6:1412
只在 `Print()` 出現一次，沒有規則。`== 1.0`（浮點相等必爛）、`>= 0.9`、`> 0.5` 都可能被寫出來。

**#15 Big-M 來源矛盾** — 見矛盾 2

**#34 Big-M 該重用既有 parameter 還是另造一顆沒指引**
fixed-charge 的最緊上界就是 `Capacity_m`，正確寫法是 `Produce ≤ Capacity_m · Open_m`；但 A.3 照字面讀會讓人新造一顆 `Parameter_BigM` 填個「夠大的數」——正是 A.3 自己警告的 M 過大。

**#40 `QTY` 是 `double`，結構常數當迴圈邊界時需要 `int`，轉型規則不存在**
§2.3 要求窗長 7、班別數做成 Parameter → 拿到 `double QTY`。`(int)`？`Convert.ToInt32`？四捨五入還是截斷？CSV 寫 `7.0` 或 `7` 有差嗎？只要照 §2.3 做就一定遇到，文件一個字都沒提。

**#46 §2.3 與 §4.4 在 `n`、`n−1` 這類值上打架**
§2.3 說結構常數用 `set.Count`，§4.4 說會變的數字 MUST 具名 PARAM。`nodes.Count` 兩者都符合又都不符合。
→ 已驗證 `set.Count` 是公開成員，建議明寫：**`set.Count` 是資料本身，可直接進係數位與常數位，NEVER 另建 `Parameter_NodeCount`**。

### 輸出與驗收契約（包 6）

**#7 §7 第 4 步「LP bound sanity」沒有可執行步驟** — §7:1461
說「整數解 ≤ LP relaxation bound」卻沒說怎麼拿到 bound。笨模型會瞎編，最糟的是把變數改成 `X_` 再跑一次（那是改模型）。
→ 明寫用 `engine.BestObjValue`，並禁止另建鬆弛模型。

**#8 例外型別與訊息格式沒規範** — §6:1406
`InvalidOperationException` + 自由格式訊息只是範例。訊息該含限制式名、索引、LHS、RHS 哪幾樣沒定。

**#9 `Print()` 完全沒契約** — §6:1410-1414
印什麼、排不排序、要不要印目標值全開放。同一題兩次產出可能完全不同。

**#10 `Solution` 類別形狀不是 MUST**
private ctor + static `ReadAndValidate` + `Print` 只是範例，沒說不能寫成 static class 或把驗證併進 Print。

**#17 驗證失敗的 exit code 未定義**
§5.1:1341 契約是「求解成功 0、失敗 1」，但 `ValidateRules` 是 throw、`Main` 沒有 try/catch → 例外穿過 `project.Execute()` 成為 unhandled，exit code 不是 1。也沒說 `OnSolved` 內的例外會不會被 `OptProject` 吞掉。該不該在 `Main` 包 try/catch？包了算不算違反「NEVER try/catch 吞例外」？

**#19 `OnSolved` 該做多少事沒界線**
範例含一次 `WriteSolution<VariableB_Assign>`。多變數型別時要不要每種都寫？§9.2.9 提供了 `ISolutionBatch` 批次，卻沒有規則說何時該用。

**#20 Logging 只有禁令沒有正面清單** — §5.1:1354
只說「不重印 framework log、只印業務語意訊息」。全檔三個 `Logging.Info` 範例格式各不相同，沒有統一契約。

**#36 `ValidateRules` 多條 constraint 時的組織方式沒規範**
§7 要求逐條代回每一條，範例只示範一條。要不要一條一個 private method、命名怎麼取、失敗要不要收集全部再一次拋（呼應框架 `DataValidationException` 的精神），三個模型會寫出三種結構。

### tuning 可執行性（包 7）

**#21 多 instance 在這套結構裡沒有路徑** — 見矛盾 4

**#22 shifted geometric mean / PAR10 只有名詞**
shift 取多少？PAR10 的 timeout 基準？在哪裡算——讀 JSON 手算還是寫成 code？若寫成 code 該放哪個檔（八資料夾裡沒有它的位置）？

**#23 promotion gate 的門檻全是形容詞**
「未達 objective / gap 品質門檻」——門檻誰定、寫在哪？「改善幅度須大於 baseline variability」——variability 怎麼量、跑幾次、取什麼統計量？沒有數字，這道 gate 不可執行，等於退回「感覺比較快就升」，而那正是整節想防的事。

**#24 champion 寫回 `productionBaseline` 沒有 before/after 範例** — §8.2:1557
`Trial.Config.SolverSpecific` 是 JSON 鍵值，要落回 C# initializer：哪些欄位寫、`null` 要不要寫、provenance 註解格式是什麼？§5 只有一行註解，沒有 promotion 後該長怎樣的樣子。這是每輪都會動到 source 的操作，卻最沒範例。

**#51 附錄 B 的「❌ 需補」是框架維護語言，開發者會誤讀** ★ 新增（範圍調整後）
附錄 B 有 10 列標「❌ 需補」（ZeroHalf 切割、Symmetry presolve、MIP start、分支優先級、自動調參、Heuristic / Lazy callback…），措辭是站在框架維護者的角度寫的。但這份文件的讀者是**模型開發者**，框架對他而言是固定的——「需補」會讓人以為可以等、可以自己加、或值得去改框架。
→ 改成能力邊界的語言：**「框架未提供，NEVER 嘗試呼叫或自行擴充；需要這類手段時回報，不要繞路。」** 並把 §5「要用得改框架本體並重編 Core + Cplex DLL」那句一併改寫——那不是開發者的選項。

**#25 `<Project>-tuning-r<N>` 的 N 怎麼續號**
規定「每輪遞增」，沒說從哪得知上一輪是幾。看 `Experiments/` 檔名（`dotnet clean` 就沒了）還是 `TuningHistory.md`？斷號或重號會直接觸發「同名 append 把歷史 trial 混進本輪」的坑。

### 其餘留白

**#3 多目標 / 加權目標式完全沒規範**
只有一句「NEVER 改權重」，但 §1.1 又寫死 `Objective/ObjectiveFunction.cs` 單一檔。加權多目標要不要拆檔、weight 從 Parameter 還是常數、要不要正規化——全空白。
→ 已由 #35 的驗證升級為**框架強制**：多目標不可能拆多檔，只能單一類別內部分段。

**#35 `CreateMinimize()` 的呼叫次數與位置沒規範** ★ 已驗證後果
`EngineBase.cs:778`：`_objectiveTerms.Clear()` → `AddRange` → `ApplyObjective()` → `ClearPool()`。呼叫兩次是**第二次靜默覆蓋第一次**，第一段目標項全部消失，照樣 build、solve、回 `Optimal`。
→ 規則：**一個模型只能呼叫一次，且 MUST 在所有目標項 `AddLHS` 完成之後**。

**#11 限制式命名只規定了 DateTime** — §4.2:1059
`int` / `double` 索引、含空白的 string 成員（`"Lot A"` → 限制式名含空白）沒規定。

**#12 非 string 維度的比較寫法**
範例一律 `p.Item == item`。`DateTime` 有無時間部分、`double` 維度用 `==` 是浮點相等陷阱，都沒講。

**#16 專案位置沒有絕對路徑** — §1.1:112
只畫了 `Projects/<Project>/`，沒說在 `$OPT/AI-Modeling/` 底下。這份文件宣稱自足，少了這句，笨模型會在當前 repo 隨手建，接著 `..\..\dlls\` 的層數就錯——正好對上 §10 的 `CS0246` 症狀。

**#18 `Export()` 名稱對齊靠人，且指向文件外的 checklist** — §2.4:810
明說「compiler 不管……MUST 在交付 checklist 逐一對數量」，但 `model-to-code-checklist.md` 不在這份自足文件裡。既是自足性破口，也等於沒有機械保證。

**#30 `Load(source, name)` 的 name 該帶哪種形式沒說**
`SetNaming.File` 會自動補前綴，所以 `"Item"` 與 `"Set_Item"` 完全等價。文件只說「MUST 顯式帶上」，沒說帶哪種，同一專案會混用。
同時暴露更基本的缺口：**類別名 / `Dataload` 欄位名 / `Load` 字串 / CSV 檔名 / 框架註冊名**五個名字的對應關係，全檔沒有一張表講完。

**#31 「Set 成員字串 PascalCase 單數」管的是資料內容，卻放在命名章節** — §1.3:164
成員值來自 CSV，是題目給的資料。真實資料是 `"TRUCK-01"` 或中文時，照這條改 CSV 就是篡改輸入，違反數值保真。沒有出口條款。

**#32 `<summary>` 三種格式沒有樣板**
Set/Parameter 是「語意；對應 Model.md 的 X」、Constraint 是「貼數學式」、Variable 是「語意 + 符號 + 值域」，三種都出現在範例裡，規則只寫「一句話寫它對應哪個符號」。

**#37 Constraint / Objective 建構子的參數順序沒規範**
只說「逐項列出實際依賴」。不影響模型，但影響 `Program.cs` call site 可讀性與 diff 穩定度，而 §5 的 chain 正是給人 review 對照用的。
→ 建議定死 **Sets → Parameters → scalars**，各組內照 Model.md 出現順序。

**#38 `.AddConstraints(...)` 的排列順序要不要對齊 Model.md 沒規範**
文件說「review 必須能從這條 chain 讀完模型組成」，但沒說順序 MUST 對齊 Model.md 的 CONSTRAINT 段。§5 自己宣稱的目的沒被規則保護。

---

## 五、審查方法與覆蓋範圍

| 輪次 | 方法 | 產出 |
| --- | --- | --- |
| 1–2 | 逐節靜態閱讀 + 弱化用語掃描 | #1–#20 |
| 3 | §8 tuning 閉環與 §1.5/§1.6 一致性 | #21–#27 |
| 4 | 交叉引用比對（22 個 §X.Y **全部指得到**）+ generator 行為比對 | #28–#32 |
| 5 | 模擬轉譯：fixed-charge 排產 | #33–#38 |
| 6 | 模擬轉譯：排班覆蓋（滾動窗、相鄰期） | #39–#44 |
| 7 | 模擬轉譯：網路流／指派（同源多角色、稀疏弧） | #45–#49 |
| 8 | 建議之間的衝突對照與收斂 | 4 矛盾 + 7 包 |
| 9 | 回框架原始碼驗證行為 | #50 + 兩項事實更正 |

**三輪模擬共掉出 17 項，全部是靜態閱讀看不到的。**

### 結構性結論

這份文件對「稠密、每個組合都有值」的題目寫得很完整，但對**稀疏結構**（哪些組合存在、哪些弧可走、哪些日期相鄰）幾乎沒有規範——而稀疏是實務題目的常態。#1、#2、#44、#45、#47、#48 其實是同一個根：**「資料的稀疏性要不要進入模型結構」沒有立場**。訂一條總則會比逐條補範例省力得多。

第二個結論：**框架有四個靜默路徑，文件只寫了零個**——漏 `partial`、漏 `: DataContext`、`Dataload` 欄位非 public、目標式 terms=0。這四個都會讓程式正常跑完並回 `Optimal`。
