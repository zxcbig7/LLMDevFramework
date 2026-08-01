# Maintenance — 制度檔案的維護協議

<system_context>
本 domain 的檔案會被之後每個 session 依賴。本協議規定：誰可以改什麼、怎麼改才安全、踩坑教訓寫到哪、累積多長要精簡。
框架在 git repo 內——git 是版本紀錄；`~/.claude/` 下的檔案不在 git，改前 MUST 先 copy 到 `~/.claude/backups/`。
</system_context>

## 分區：什麼可以自己改

| 區 | 檔案 | 規則 |
|---|---|---|
| 🟢 綠區（自行改，回報一句） | `lessons.md`（追加）、`templates/*.md` 補充範例與填空要點 | 追加不刪改既有內容；改完 read-back |
| 🟡 黃區（自行改，但走下方流程） | `dispatch.md`、`judgment.md`、`orchestration/CLAUDE.md`、agent 定義的 prompt 本文 | 改前備份、改後派審，見「黃區流程」 |
| 🔴 紅區（先問使用者才能動） | `~/.claude/CLAUDE.md`、`router/global-claude-block.md`、agent 定義的 `model`/`effort` frontmatter（成本）、`bootstrap/inventory.json`、刪除任何檔 | 列 diff + 動機 + 影響面，等使用者點頭 |
| ⛔ 凍結（NEVER 改） | `diagnosis-2026-07-08.md`、`LETTER.md` | 歷史文件；狀況變了就在 `lessons.md` 記「已過時」條目，不改原文 |

## 黃區流程（缺一步不算完成）

1. 備份：git repo 內的檔案靠 git（改前確認 working tree 乾淨）；`~/.claude/` 下的先 copy 到 `~/.claude/backups/`
2. 改動遵守：`prompt-principles/CLAUDE.md` 的 self-check、NEVER/ALWAYS/Why 三段式、單檔 ≤200 行
3. 超過 200 行 → 抽引用檔，主檔留路由——NEVER 靠刪 Why 來瘦身（Why 是弱模型的邊界判斷力來源）
4. 派 fresh-context 審查（`templates/review.md`，維度加一條「與 dispatch/judgment 其他條文的相容性」）
5. 改的是 agent 定義 → 同步重新部署：copy 到 `~/.claude/agents/` 同名檔（重開 session 生效）
6. 更新 `$FW/CodeMap.md` File Index 的行數欄——僅當 `orchestration/CLAUDE.md` 本身行數變動時（File Index 只登錄該檔；`/harness-doctor` 檢查 5 會抓）

## 踩坑教訓寫回哪裡

**觸發**：以下任一發生 → 當個 session 內就寫，不要「之後再記」（session 結束就忘了）：
- 同一子任務重試 2 輪才過 / 換路才解
- verifier 抓到你以為完成的 FAIL
- 使用者糾正了你的做法
- 規則照著做卻出錯（規則本身有坑）

**寫到**：`orchestration/lessons.md`，append-only，格式見該檔開頭。
**提升**：同型教訓出現第 2 次 → 把它改寫成 `dispatch.md` 或 `judgment.md` 的正式條文（黃區流程），
並在 lessons 條目補 `→ 已提升：<檔>§<節>`。單次教訓 NEVER 直接進守則——一次可能是雜訊，兩次才是 pattern。

## 精簡規則（防制度肥大）

- `lessons.md` 超過 30 條或 150 行 → 做一輪精簡：已提升的刪本文留一行索引、同型合併、超過 6 個月且再沒發生的刪除
- `dispatch.md` / `judgment.md` 各 ≤200 行是硬上限；要加新條文而超限 → 先刪一條最少用到的（刪誰拿不準 → 問使用者）
- 每季（或使用者說「檢查制度」時）跑一次：`/harness-doctor` + 把 orchestration 各檔互相引用的路徑全 Glob 驗證一次

## 判斷不了的時候

改動屬於哪一區拿不準 → 當作高一級處理（綠當黃、黃當紅）。
改了會不會跟別的條文打架拿不準 → 派 `second-opinion` 裁決，仍不確定 → 問使用者。
