# LLMDevFramework

<system_context>
LLM 開發框架與規範庫，存放各語言/技術棧的 Claude Code 開發守則。
子目錄依技術分類（OracleSQL、React & Typescript），每個子目錄可有自己的 CLAUDE.md。
</system_context>

<critical_notes>

- MUST 寫 / 改任何 CLAUDE.md 或 slash command 前先讀 `prompt-principles/CLAUDE.md`，寫完跑 self-check
- MUST 讀任何 domain CLAUDE.md 或做框架級改動後，同步更新 root `CodeMap.md`（File Index 行數 + Coverage Assessment）
  Why: CodeMap 是跨 session 的導覽地圖，不更新就會過時，下次開發還是要從頭找
- MUST 改任何有 `evals/` 的 domain CLAUDE.md 後跑 `/harness-eval <domain>` 驗證行為
  Why: 沒 eval 的規範改動是憑感覺，無 regression 檢測（框架的 unit test）
- 規範文件用 terse but complete 風格，100–200 行為上限
- 規則用字：MUST / NEVER / ALWAYS；重要規則用「NEVER X — ALWAYS Y — Why: Z」三段式
- DRY：用 file reference / 巢狀 CLAUDE.md，不要重複內容
- 預設用繁體中文，technical terms 保留英文
- 給範例優先用 good/bad 對照（multi-shot），純文字描述次之

</critical_notes>

<file_map>
prompt-principles/      - 元規範：寫 CLAUDE.md / slash command 的 12 prompt 技巧（所有新文件先看這）
Prompt Builder/         - 寫好 prompt 的工具箱（10 個框架 + `/prompt-improve` 指令 + 模板）
OracleSQL/              - Oracle SQL 開發規範與 pattern
React & Typescript/     - React + TypeScript 前端開發規範
.Net Web API/           - ASP.NET Core Web API 後端開發規範
YAML Review/            - YAML review 規範（K8s、Helm、ArgoCD、GitHub Actions、Docker Compose）
sdd/                    - Spec-Driven Development 方法論 + slash command + 規格模板
karpathy-guidelines/    - Karpathy 四原則守則（Think Before Coding / Simplicity / Surgical / Goal-Driven）+ `/kg` slash command
Slide Builder/          - 投影片製作 orchestrator：依需求路由 pptx（可編輯）/ guizang HTML（網頁 deck）+ brand kit
Mermaid Diagrams/       - 專業 Mermaid 製圖規範 + `mermaid-diagrams` skill：base theme + 語義 classDef 美化、語法防呆、mmdc/Kroki 匯出
CMD Developer/          - Windows batch `.bat` 開發規範 + `/cmd-dev`（9 大雷區）
PowerShell/             - PowerShell `.ps1` 開發規範 + `.bat` vs `.ps1` 選用決策（與 CMD Developer 並列）
router/                 - 自適應分派層：Router 區塊注入全域 CLAUDE.md（語言→domain + 任務→skill）+ `/scaffold` 半自動產專案 CLAUDE.md
harness-eval/           - 框架行為評測（behavioral eval）：case 模板 + `/harness-eval` runner——框架的 unit test
bootstrap/              - 可攜層：inventory.json 部署清單 + `/harness-doctor` 三方對帳；AI 適配入口在 root `BOOTSTRAP.md`
code-review/            - `/code-review` source：CodeMap 前置 + 獨立 evaluator subagent review
MILP Model/             - MILP 數學模型開發 domain（服務 OptimizationFramework / OptimFoundation CPLEX）：三階段天條 + 三 phase 子規範 + evals
workflow skill/         - milp-dev skill：三階段 phase gate orchestrator + status.json resume（規範單一來源在 MILP Model/）
</file_map>

<paved_path>
- 新增技術領域 → 建立子目錄 + 子目錄內 CLAUDE.md
- 子 CLAUDE.md 只放該技術專屬規則，通用規則留在本檔
- 範例 / pattern 用 search term 指引位置，不直接貼大段 code
</paved_path>

<patterns>
規範文件結構（每個子目錄 CLAUDE.md 建議）：
- system_context  - 技術棧用途
- critical_notes  - 硬性規則
- file_map        - 子目錄結構
- paved_path      - 標準作法
- patterns        - 常見 pattern + 位置
- common_tasks    - 任務指引
- fatal_implications - 絕對禁止
</patterns>

<workflow>
## 新功能開發流程（SDD - Spec-Driven Development）

所有非 trivial 新功能 MUST 走這條路徑（細節見 `sdd/CLAUDE.md`）：

1. **Plan mode 討論**：抓出方向 + 技術選型
2. **`/sdd` 釐清**：Claude 問三題（一句話描述、互動模組、成功標準）
3. **產出規格文件**：存到當前專案 `specs/YYYY-MM-DD-<slug>.md`
4. **產 `CodeMap.md`**：掃描受影響模組，畫 Dependency Graph，確認規格與現有架構不衝突 —— MUST 在 stub 前完成
5. **空殼實作（stub）**：先把 interface / 函式簽名 / 路由建好，邏輯留 `TODO`
6. **逐段實作 + 對照規格驗收**

## 維護規範流程

1. 發現新 best practice / anti-pattern
2. 判斷屬於哪個技術領域 → 寫進對應子 CLAUDE.md
3. 跨領域通用規則 → 寫進本檔
4. 過時內容直接刪除，不留 deprecated 註解
</workflow>

<common_tasks>
- 新增 pattern → 找對應子目錄 CLAUDE.md 的 patterns 區塊
- 新增技術棧 → 建子目錄 + CLAUDE.md（複製本檔結構）
- 引用範例 code → 用 `path/to/file.ts`, search:`symbolName`
</common_tasks>

<example>
file reference 寫法：
- Custom hook → `React & Typescript/hooks/useX.ts`, search:`useDeferredValue`
- PL/SQL package → `OracleSQL/packages/PKG_X.sql`, search:`PROCEDURE foo`
- Controller pattern → `.Net Web API/Controllers/XController.cs`, search:`HttpGet`
</example>

<hatch>
- 規則衝突時 → 子目錄 CLAUDE.md 優先於本檔
- 特殊專案需求 → 在該專案 root 另開 CLAUDE.md 覆寫
</hatch>

<fatal_implications>
- NEVER 把 secrets / 連線字串寫進規範文件
- NEVER 在規範裡貼整份檔案內容（用 search term 引用）
- NEVER 讓單一 CLAUDE.md 超過 200 行
</fatal_implications>
