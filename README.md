# LLMDevFramework

個人化的 **Claude Code 開發框架**——把跨專案常用的開發守則、工作流程、slash command、skill 集中在一個 repo，讓 AI 在任何專案都能依照一致的標準產出，並且**可驗證、可對帳、可回歸測試**。

> **Why**：Claude 每個 session 重新開始，沒有記憶。把規範寫進 CLAUDE.md / slash command，等於把你的開發判斷外包給 AI 卻不失控；再用 eval 與 doctor 確保規範真的被遵守、部署真的沒漂移。

---

## 快速開始

### 1. Clone

```powershell
git clone <repo-url> C:\Users\<you>\Desktop\Projects\LLMDevFramework
```

### 2. AI 適配（取代安裝腳本）

開 Claude Code，對它說一句：

> **讀 BOOTSTRAP.md，完成適配**

AI 會自己跑完四個 phase：

| Phase | 動作 |
| --- | --- |
| 0 盤點 | 讀 `bootstrap/inventory.json` 部署清單，盤點這台機器的 `~/.claude/` 現況 |
| 1 計畫 | 列「將寫 / 將改 / 跳過」三欄計畫表，**等你確認才動手** |
| 2 執行 | 寫入 commands、skills、Router 區塊、manifest（`~/.claude/.llmdevframework.json`） |
| 3 驗證 | 用 `/harness-doctor` 七項檢查自我驗收 |

重跑安全（idempotent）。`git pull` 拉了框架更新後，再說同一句即可增量更新。

### 3. 日常使用

適配完成後**不需要手動做任何事**。Router 區塊已注入全域 `~/.claude/CLAUDE.md`，在任何專案工作時 Claude 會自動：

- 看到 `.sql` → 讀 OracleSQL 規範；`.tsx` → 讀 React & TS 規範；`.ps1` → 讀 PowerShell 規範……
- 非 trivial 新功能 → 走 `/sdd`；模糊任務 → `/kg` pre-flight；「幫我看 code」→ `/code-review`
- 進到沒有 CLAUDE.md 的新專案 → 主動建議 `/scaffold`

### 對帳 / 更新 / 移除

- 懷疑裝的東西過時或缺件 → `/harness-doctor`（read-only，回報 drift + 修法）
- 更新 → `git pull` 後說「讀 BOOTSTRAP.md，完成適配」
- 移除 → 對 Claude 說「依 `~/.claude/.llmdevframework.json` manifest 移除框架部署的檔案」

詳見 [`BOOTSTRAP.md`](./BOOTSTRAP.md) 與 [`bootstrap/CLAUDE.md`](./bootstrap/CLAUDE.md)。

---

## 架構

框架分五層，`CodeMap.md` 是跨 session 的導覽地圖（File Index + Dependency Graph + Coverage Assessment）。

```text
LLMDevFramework/
├── README.md # 本檔
├── CLAUDE.md # 框架根規範（維護準則 + SDD 流程）
├── CodeMap.md # 框架地圖（File Index + Mermaid 依賴圖 + Coverage）
├── BOOTSTRAP.md # ★ AI 適配安裝 playbook（取代安裝腳本）
├── teck.md # Claude Code 表現提升手法總整理
│
│ # ─ 基礎設施層 ─
├── bootstrap/ # 可攜層：inventory.json 部署清單（single source）+ /harness-doctor 三方對帳
├── router/ # 自適應分派：Router 區塊注入全域 CLAUDE.md + /scaffold + 專案模板
├── harness-eval/ # 框架行為評測（behavioral eval）：case 模板 + /harness-eval runner
│
│ # ─ 元規範層 ─
├── prompt-principles/ # 寫任何 CLAUDE.md / command 前必讀：12 prompt 技巧 + self-check
│
│ # ─ 工作流層 ─
├── sdd/ # Spec-Driven Development：/sdd + 規格模板
├── karpathy-guidelines/ # Karpathy 四原則 pre-flight：/kg
├── code-review/ # /code-review：CodeMap 前置 + 獨立 evaluator subagent
├── Prompt Builder/ # 寫好 prompt 的工具箱：10 框架 + /prompt-improve + 模板
│
│ # ─ Domain 規範層 ─
├── OracleSQL/ # PL/SQL 規範 + proc-analysis/（/proc-analyze + 筆記庫）
├── React & Typescript/ # React + TS strict 規範 + paved_stack + frontend-resources + evals/
├── .Net Web API/ # ASP.NET Core Web API 規範
├── YAML Review/ # K8s/Helm/ArgoCD/GHA：/k8s-review + troubleshooting/ 經驗庫
├── CMD Developer/ # Windows batch .bat 規範 + /cmd-dev（9 大雷區）+ evals/
├── PowerShell/ # .ps1 規範（BOM/CRLF/5.1 vs 7+）+ .bat vs .ps1 選用決策 + evals/
├── MILP Model/ # MILP 數學模型開發：三階段天條 + Model Design / Foundation Coding / Foundation Tuning + evals/
│
│ # ─ 工具 skill 層 ─
├── Mermaid Diagrams/ # mermaid-diagrams skill：base theme + 語義 classDef + 匯出
├── Slide Builder/ # slide-builder skill：路由 pptx（可編輯）/ guizang HTML（網頁 deck）
├── workflow skill/ # milp-dev skill：MILP 三階段 phase gate orchestrator + resume
│
└── specs/ # 框架自身功能的 SDD 規格文件（dogfooding）
```

### 依賴關係（簡化）

```text
prompt-principles ← 所有 CLAUDE.md / command 的寫作依據
router → 注入全域 CLAUDE.md → 自動 dispatch 各 domain 規範
bootstrap/inventory.json → BOOTSTRAP.md 部署 → /harness-doctor 對帳
harness-eval → 對有 evals/ 的 domain（CMD / PowerShell / React&TS）做回歸測試
code-review → 每次 review 先產 CodeMap.md
```

完整版見 [`CodeMap.md`](./CodeMap.md) 的 Mermaid Dependency Graph。

---

## 內建工具

### Slash Commands（10 個）

| Command | 用途 | Source |
| --- | --- | --- |
| `/sdd <一句話描述>` | Spec-Driven Development 啟動器：問 3 題 → 產規格 → approve → stub → 逐段實作 | `sdd/` |
| `/kg <任務>` | Karpathy 四原則 pre-flight：把模糊任務轉成清晰執行計畫 | `karpathy-guidelines/` |
| `/code-review` | 先產 CodeMap.md 當地圖，再交獨立 evaluator subagent 依圖 review，findings 經主 session 驗證才輸出 | `code-review/` |
| `/prompt-improve <草稿>` | 5-step 改造：品質診斷 → 釐清 → 選框架（CO-STAR / RISEN / TIDD-EC…）→ XML 重寫 → 對照表 | `Prompt Builder/` |
| `/k8s-review <路徑>` | 無 kubectl 環境的 K8s YAML 靜態 audit：11 維度、CRITICAL/WARN/INFO 分級、吃 troubleshooting 經驗庫 | `YAML Review/` |
| `/proc-analyze <檔案>` | 10K+ 行 PL/SQL 多 pass 分析：grep 骨架 → 分段填肉 → 3 張 Mermaid → 筆記庫 | `OracleSQL/proc-analysis/` |
| `/cmd-dev <需求>` | Windows `.bat` 寫 / review / debug 三合一（9 大雷區防呆） | `CMD Developer/` |
| `/scaffold` | 偵測專案技術棧，半自動產生 reference 框架的 CLAUDE.md（root + 子資料夾） | `router/` |
| `/harness-eval <domain>` | 行為評測 runner：spawn 看不到答案的被測 subagent，對照 case 判準回報 PASS/FAIL/STALE | `harness-eval/` |
| `/harness-doctor` | 框架三方對帳（source ↔ inventory ↔ `~/.claude`）：七項檢查，read-only | `bootstrap/` |

### Skills（3 個 + 外部依賴）

| Skill | 用途 |
| --- | --- |
| `mermaid-diagrams` | 產專業不醜的 Mermaid 圖：base theme + themeVariables、語義 classDef 上色、語法防呆、mmdc/Kroki 匯出 |
| `slide-builder` | 投影片 orchestrator：讀素材 + brand kit，路由到 pptx skill（可編輯 .pptx）或 guizang-ppt-skill（網頁 deck） |
| `milp-dev` | MILP 建模 orchestrator：三階段 phase gate（Model Design → Foundation Coding → Foundation Tuning）+ status.json resume，服務 OptimizationFramework（CPLEX） |

外部依賴（不由本框架部署）：`guizang-ppt-skill`（自行 git clone）、Anthropic `pptx` skill（自 anthropics/skills 複製）。

---

## 核心工作流教學

### 新功能開發（SDD）

所有非 trivial 新功能 MUST 走這條路（細節見 [`sdd/CLAUDE.md`](./sdd/CLAUDE.md)）：

1. **Plan mode 討論**：抓方向 + 技術選型
2. **`/sdd` 釐清**：Claude 一次問 3 題（一句話描述、互動模組、成功標準）
3. **產規格**：存 `<project>/specs/YYYY-MM-DD-<slug>.md`，等 approve
4. **產 `CodeMap.md`**：掃受影響模組、畫依賴圖，確認規格不與現有架構衝突——MUST 在 stub 前完成
5. **Stub**：interface / 函式簽名 / 路由先建好，邏輯留 `TODO`
6. **逐段實作 + 對照規格驗收**

### 改框架規範（eval loop）

規範不是寫完就算——有 `evals/` 的 domain（CMD Developer、PowerShell、React & TS，各 3 cases）改動後 MUST 跑回歸：

1. 改 domain `CLAUDE.md`
2. `/harness-eval <domain>`：runner spawn 一個看不到答案的被測 subagent 做任務，對照 case 判準評 PASS / FAIL / STALE
3. FAIL → 規範沒寫到位，回頭修；STALE → case 的 rule_refs 錨點失效，更新 case
4. 同步更新 `CodeMap.md`（File Index 行數 + Coverage Assessment）

### 部署健檢（doctor）

`/harness-doctor` 做三方對帳：**source（本 repo）↔ inventory（部署清單）↔ `~/.claude/`（實際部署）**，七項檢查涵蓋漏裝、孤兒、內容 drift、Router 區塊完整性。Read-only，只報 drift + 修法，不動手改。

### Code Review

`/code-review`（或說「幫我看 code」）流程：先產 / 更新 `CodeMap.md` → 交給獨立 evaluator subagent 依圖 review（避免自審自誇 + 重複瀏覽耗 token）→ findings 經主 session 驗證才輸出。**NEVER 在 CodeMap 產完前輸出 review 評語。**

---

## 設計理念

1. **XML semantic tags 結構化**：每份 CLAUDE.md 套相同骨架——`system_context` / `critical_notes` / `file_map` / `paved_path` / `patterns` / `common_tasks` / `example` / `hatch` / `fatal_implications`
2. **100–200 行上限**：CLAUDE.md 每 turn 都載入，超過 200 行會稀釋 instruction adherence
3. **12 prompt 技巧檢核**：寫 / 改任何 CLAUDE.md 或 command 前先讀 [`prompt-principles/CLAUDE.md`](./prompt-principles/CLAUDE.md)，寫完跑 self-check
4. **Multi-shot 對照優先**：good/bad code 對照 > 純文字描述，Claude 學範例學得很細
5. **經驗累積 > 一次寫死**：`troubleshooting/`、`proc-analysis/notes/` 空殼起手，撞到問題才填，隨時間長成個人知識庫
6. **Eval-driven 規範維護**：規範改動要有 behavioral eval 當回歸測試，不憑感覺
7. **AI 適配 > 安裝腳本**：部署交給 AI 讀 playbook 執行（可解釋、可確認、跨 OS 自適應），`scripts/` 已於 2026-07-04 廢棄刪除
8. **規則三段式**：重要規則寫成「NEVER X — ALWAYS Y — Why: Z」

---

## 如何擴充

### 加新技術領域

1. 建子資料夾（例如 `Python/`），複製其他子目錄的 CLAUDE.md 結構（XML 9 tag）
2. 在 root [`CLAUDE.md`](./CLAUDE.md) `<file_map>` 補路徑
3. 在 [`router/global-claude-block.md`](./router/global-claude-block.md) 加 dispatch 規則（偵測 → Read）
4. 更新 `CodeMap.md`；建議同步寫 2–3 個 `evals/` case

### 加新 slash command

1. 寫 `<domain>/<name>-command.md`，**必含**：frontmatter（`description` + `argument-hint`）、`<role>` 開場、步驟編號 + CoT 觸發、good/bad 範例、`<output-format>` pre-fill、`<rules>` 結尾
2. 對照 [`prompt-principles/CLAUDE.md`](./prompt-principles/CLAUDE.md) self-check 跑一次
3. 登錄 [`bootstrap/inventory.json`](./bootstrap/inventory.json)（部署清單 single source）
4. 對 Claude 說「讀 BOOTSTRAP.md，完成適配」增量部署

### 加新經驗 case

撞到問題後 30 分鐘內寫——記憶最熱：

- K8s 部署坑 → 複製 `YAML Review/troubleshooting/_template.md`
- PL/SQL procedure → 跑 `/proc-analyze` 自動產筆記
- 規範行為 case → 複製 `harness-eval/case-template.md` 放進該 domain `evals/`

---

## 維護規則

- **可重現的東西不寫進 memory**：code 結構、git history、檔案路徑都從 repo 直接讀
- **過時內容直接刪**：不留 deprecated 註解
- **規範衝突優先序**：專案 CLAUDE.md > 子目錄 CLAUDE.md > 框架 root CLAUDE.md
- **改動後同步 `CodeMap.md`**：File Index 行數 + Coverage Assessment，不然地圖過時形同沒有
- **部署清單只改 `bootstrap/inventory.json`**：NEVER 手動改 `~/.claude/` 下的部署檔

---

## 參考資源

完整 sources 列表見 [`teck.md`](./teck.md) 末尾。核心參考：

- [Anthropic Claude 4 Prompt Engineering 12 技巧](https://codelove.tw/@tony/post/3189Kx)
- [How I Use Claude Code (Tyler Burnam)](https://tylerburnam.medium.com/how-i-use-claude-code-c73e5bfcc309)
- [SDD Skill (wu_pingju)](https://www.threads.com/@wu_pingju/post/DTiQ8dWFHlw)
- [GitHub spec-kit](https://github.com/github/spec-kit)
- [Anatomy of the .claude/ Folder](https://blog.dailydoseofds.com/p/anatomy-of-the-claude-folder)

---

## License

個人使用框架，無 license。
