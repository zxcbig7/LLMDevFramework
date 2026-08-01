# Lessons — 踩坑教訓登記簿（append-only）

> 格式（一條 5 行內，寫完就走）：
>
> ```text
> ## YYYY-MM-DD <一句話標題>
> 情境：當時在做什麼
> 錯誤：實際發生什麼（含證據：檔案 / 指令 / 輸出）
> 教訓：下次遇到同情境怎麼做
> 狀態：未提升 ／ → 已提升：<檔>§<節>
> ```
>
> 規則見 `maintenance.md §踩坑教訓寫回哪裡`：同型第 2 次出現才提升為正式條文；超過 30 條或 150 行做精簡輪。

## 2026-07-08 部署後的檔案繞過 source 改動必 drift

情境：Fable session 重寫全域 CLAUDE.md 前，比對 `~/.claude/CLAUDE.md` 的 Router 區塊與 source `router/global-claude-block.md`。
錯誤：部署版缺了 source 已有的 MILP 兩列——先前某次只改了 source（或只改了部署版），沒有同步另一邊。
教訓：動「有 source ↔ deployed 兩份」的檔案（router 區塊、commands、skills、agents），MUST 改 source 再重新部署，NEVER 直接改部署版；改完跑 `/harness-doctor` 對帳。
狀態：未提升（本來就是 bootstrap 既有規則，記錄實證一次）

## 2026-07-18 假物件自己模擬行為 → 測試全綠但真實作零覆蓋

情境：OptimFoundation 資料防護規格，Block 4 交付「8 支測試驗證 DB transaction 回滾」。
錯誤：測試全部只用 `FakeDbCtrl`，而該假物件**自己**用 try/catch 模擬 commit/rollback；驗收時把真實作 `OracleDBCtrl.ExecuteInTransaction` 的 `tx.Rollback()` 註解掉 → **132 支測試仍全過**。測到的是假物件自己的邏輯，真實作零覆蓋。
教訓：交付含「行為測試」時，MUST 做**反向證明**——把被測的那行真實作破壞掉，確認**有測試會紅**；不紅就是綠燈假象。派工單直接寫進驗收條件（要求執行者自己先做並附失敗測試名稱），比事後靠 verifier 抓便宜。修法通常是「把與外部相依無關的編排邏輯下沉到可測的層」（本例：交易編排移到 `DBCtrlBase`，用 BCL `IDbConnection`/`IDbTransaction` 承載，假連線驅動真邏輯）。
狀態：未提升（同型第 2 次出現再提升為 `dispatch.md §驗證不自驗` 正式條文）

## 2026-07-18 git diff 看不到 gitignore 的檔，殘留與過期產物會躲過驗收

情境：同上規格。多輪 verifier 都以 `git diff` / `git status` 為空來證明「破壞測試已還原、無殘留」「dll 已更新」。
錯誤：(1) 破壞測試留下的 `Parameter_Demand.csv.bak` 被 gitignore，三輪檢查都沒發現；(2) 結案時發現 `Templates/dlls/*.dll`（同樣 gitignore）比原始碼舊兩小時、不含當波所有改動——原始碼綠、測試綠，但實際部署的 dll 是舊的。
教訓：驗「殘留」與「部署產物是否最新」時，`git` 幫不上忙。MUST 改用 `find` 掃備份類副檔名（`*.bak`/`*.orig`/`*.tmp`），並用**檔案時間戳或型別存在性**（如 `grep -a <新型別名> <dll>`）驗證部署產物；派工單要明寫這條，否則 subagent 預設會用 git 檢查。
狀態：未提升

## 2026-07-19 文件汰舊換新只改「被點名的檔」，會製造比原本更危險的自我矛盾

情境：框架 API 改版後，使用者點名要更新 AI-Modeling 的「Code 轉譯與 tune 任務 prompt」。我依字面把範圍限縮在 Prompts / tuning 這批檔。
錯誤：跨輪一致性驗收揪出三處殘留矛盾，全在我劃出的範圍外——(1) `Template_CPLEX/CLAUDE.md` 標題寫「預設：source generator」底下卻教舊字串式，而該檔自稱 canonical 範本來源；(2) `CPLEX_API_REFERENCE.md` 被我們改了 §7.6 卻沒改 §14/§16.2 的「完整最小範例」，**同一份文件自己打架**——這比整篇都舊更糟，讀者無從判斷該信哪段；(3) 注入型檔案 `automated/CLAUDE.md` 標題明寫「注入每個 Prompt」，它若沒改會在執行期把 AI 拉回舊寫法，讓前面 18 檔的修正失效。前幾輪 agent 都把這些列在「額外發現」回報過，是我判斷成範圍外沒接住。
教訓：文件不是獨立檔案而是**有引用與繼承關係的系統**。汰舊換新前 MUST 先畫出關係圖並問三題：(a) 有沒有**注入型/被 include** 的上游檔（改了下游沒改上游＝白改）；(b) 有沒有**自稱權威/canonical** 的檔（源頭錯＝全部錯）；(c) 同一份檔內有沒有**多處講同一件事**（只改一處＝自我矛盾）。改一半比不改危險。
狀態：未提升

## 2026-07-19 派工單的驗收條件寫錯層級／已知坑沒同步到所有相關單子

情境：OptimFoundation 資料防護規格收尾，連續派多個 executor 做範本 migration。
錯誤：(1) 驗收條件寫「`git diff -- src/` 必須為空」，但 `src/` 有整個 session 未 commit 的框架成果，**這條永遠不可能滿足**——agent 要嘛卡住不敢收工、要嘛只能謊報 PASS，兩種都是派工方造成的。實際結果：agent 回了一句「正在等待背景執行完成」就停住，沒交報告。(2) 已知某範本尾端有 7 trial × 180 秒的 benchmark，我只把「用 timeout 包住」寫進**下一張**派工單，沒寫進**同樣會碰到該範本**的另一張 → 第二個 agent 又卡在同一處。
教訓：(a) 驗收條件 MUST 針對「**這次派工的行為**」（你有沒有動 X），NEVER 針對「**repo 的全域狀態**」（X 乾不乾淨）——多波次進行中的工作，全域狀態本來就不乾淨。送出前自問一次「以現在的 repo 狀態，這條可能被滿足嗎」。(b) 踩過的坑要寫進**所有會碰到它的**派工單，不是只寫下一張；長時間執行的驗證一律附 `timeout` 用法與「只取哪幾行輸出」。
狀態：未提升（同型第 2 次出現再提升為 `dispatch.md §派工三件套` 正式條文）

## 2026-07-18 驗證掛在哪一層迴圈，決定它有沒有靜默洞

情境：同上規格，`DataValidator` 的參照完整性檢查。
錯誤：「index-set 名不存在」的偵測寫在**逐列迴圈內**，於是「零列 parameter + set 名打錯」時迴圈根本不執行＝沒人報；而下游的 `[FullGrid]` 檢查遇到找不到 set 也直接 return，兩邊互相以為對方會處理，形成完全靜默。附帶問題：有 N 列時噴 N 筆重複噪音。
教訓：驗證「宣告層級的東西」（schema、名稱、型別）MUST 掛在該層級跑一次，NEVER 掛在資料列迴圈裡——否則空資料集會讓驗證整個消失。設計檢查鏈時，明確寫下「A 檢查跳過時由誰負責回報」，別讓兩個檢查互推。
狀態：未提升

情境：盤點 harness 設定。
錯誤：`settings.json` 的 `"model": "claude-fable-5[1m]"` 在 Fable 停用後會失效；寫死完整 model id 的設定有壽命。
教訓：settings 的 model 欄位優先用別名（`sonnet` / `opus` / `opusplan`），別名隨 Claude Code 版本自動指向最新；寫死完整 id 只在需要鎖版本時用，且要記錄解鎖條件。
狀態：未提升
