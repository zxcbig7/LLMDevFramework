# 給未來 session 的信

> 寫於 2026-07-08，Fable 5 的一次性 session。這個 session 之後，這個環境由 Sonnet / Opus / Haiku 長期運作。
> 本檔凍結（`maintenance.md §分區`）：狀況變了在 `lessons.md` 記條目，不改本文。

## 三件沒被問到、但對這個環境最重要的事

### 1. 有一批設定殘留需要使用者決定，越早處理越好（紅區，AI 不可自行動手）

细節與證據在 `diagnosis-2026-07-08.md §Top-2`，濃縮成待決清單：

| 待決 | 建議 |
|---|---|
| `settings.json` `"model": "claude-fable-5[1m]"` 已失效 | 改 `"opusplan"`（規劃 opus、執行 sonnet）或 `"sonnet"`；`effortLevel: xhigh` 若日常嫌貴降 `high` |
| `mcp.json` 的 `notion` server 是死的（`Bearer AAA` placeholder） | 刪 entry——claude.ai 的 Notion connector 已提供同功能 |
| `C:\Users\zxcbi\CLAUDE.md`（4 行，與全域重複） | 刪除（內容已併入全域 CLAUDE.md） |
| `additionalDirectories` 含整個 home dir | 收斂成實際需要的子目錄 |
| `milp-dev` skill 在 inventory 有、`~/.claude/skills/` 沒有（既有 drift，2026-07-08 對抗審查抓到） | 重跑 BOOTSTRAP Phase 1–2 補裝 |

第一個接手的 session：把這張表念給使用者聽，拿到決定就執行。執行完在 `lessons.md` 記一筆，並在本節表格加一欄「✅ 已處理 YYYY-MM-DD」（這是唯一允許動本檔的例外）。

### 2. 制度的生效機制是「迴路」，不是「規則」

這個框架已經有三個迴路：`/harness-doctor`（結構對帳）、`/harness-eval`（行為回歸）、`verifier` agent（交付驗收）。
規則會被長 context 沖掉（`diagnosis §Top-3`），迴路不會——因為迴路是在「事後」用 fresh context 檢查，不依賴執行者記得。

所以：想讓某件事被穩定遵守，**不要加規則，加檢查**。
orchestration domain 目前還沒有 `evals/`——第一次要改 `dispatch.md` 或 `judgment.md` 之前，先照 `$FW/harness-eval/` 的 case 模板建至少 2 個 case（建議：「派工缺驗收條件時 verifier 是否拒收」「主對話遇掃 repo 任務是否派 Explore」），改完跑 `/harness-eval orchestration`。

### 3. 三個記憶系統的分工（寫錯地方 = 下次找不到）

| 系統 | 範圍 | 該放什麼 |
|---|---|---|
| auto-memory（`~/.claude/projects/<project>/memory/`） | **單一專案** | 該專案的進行中狀態、使用者對該專案的偏好 |
| `$FW/orchestration/lessons.md` | **跨專案、關於工作方法** | 踩坑教訓、制度實證 |
| CLAUDE.md 疊層（全域 → 專案） | **常駐規則** | 只放每個 session 都需要的；細則抽檔路由 |

判準：這件事換一個專案還成立嗎？成立 → lessons.md；不成立 → 該專案 memory。
NEVER 同一件事寫兩處——會分岔，之後兩個版本互相矛盾。

## 這套制度最可能的退化方式，與預防法

1. **規則堆積**：每次踩坑加一條，兩年後 500 行沒人讀得完。
   防線：`maintenance.md` 的 200 行硬上限 + 「同型第 2 次才提升」+ 精簡輪。加新條文前先問：能不能用既有條文 + 一個範例表達？
2. **儀式化**：形式上照做、實質失效——派了 verifier 但驗收條件寫「功能正常」；產了 CodeMap 但 review 根本沒引用它。
   防線：`verifier` 的 input-contract 會拒收無驗收條件的派工（設計進 agent 定義，不靠自覺）；發現任何「跑了流程但沒攔到問題」的案例 → lessons.md 記下來，那是流程的 bug。
3. **同步斷裂**：改 source 不部署、改部署版不改 source（本 session 就實地抓到一次：router 區塊缺 MILP 列，見 `lessons.md` 第一條）。
   防線：定期 `/harness-doctor`；「有兩份的檔案只改 source」。
4. **靜默繞過**：趕時間「這次先不派驗收」，一次沒事就變常態。
   防線：`dispatch.md §Fallback` 要求繞過必須標明 `⚠️ 未派工`——讓繞過對使用者可見。使用者看到標記變頻繁，就是制度在退化的警報。

## 交接狀態（2026-07-08 session 結束時更新）

- A–G 七項交付全部落檔於 `$FW/orchestration/`；全域 CLAUDE.md 已重寫（備份在 `~/.claude/backups/fable-2026-07-08/`）
- agents 已部署 `~/.claude/agents/`（verifier / second-opinion / batch-worker）——**重開 session 才生效**
- 未完成項目：見本檔 §1 的待決清單（需使用者決定，非技術債）
