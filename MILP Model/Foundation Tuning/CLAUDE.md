# Foundation Tuning — Phase 3 調校規範

<system_context>
模型通過正確性 gate 後才可調校；使用者提出才做。完整旋鈕見 `$OPT/AI-Modeling/tuning/CLAUDE.md`；手動掃描仍不達標或要評估自動調參工具時，讀同層 `solver-tuning-research.md`。
</system_context>

<critical_notes>

- 先驗正確，再調效能。
- 依 solver / data / structure 三類觸發走對應入口，NEVER 混路。
- 模型已定版後，「太慢 / gap 下不去」先試可逆的 `CplexConfig` 旋鈕，連續 3 輪無改善才升級到 structure；建模當下的模型優先原則不能拿來跳過 solver 層。
- 每輪使用 `OptExperiment` 記錄 objective、MipGap、時間、節點數與 Status，回報 before / after。
- tuning MUST 形成 AI 閉環：產生 variants → 跑實驗 → 讀本輪 Trial → 選 champion → 通過 promotion gate → 更新 `Program.cs` 的 production baseline → 重跑 production 驗證。只跑 experiment、不把勝出設定升級到 production，視為尚未完成 tuning。
- production baseline 是 `Program.cs` 中唯一一顆具名 `productionBaseline`；experiment 只能從它 `Clone()`，prod 的 `OptProject.UseConfig` 也 MUST 使用它。NEVER 維護「實驗 baseline」與「production config」兩份會漂移的設定。
- 每次 promotion MUST 有永久 provenance：同步更新 baseline 上方註解與專案根 `TuningHistory.md`，寫明來源 experiment、champion Trial、before/after config diff、評分證據及 production 驗證。只留在 `bin/Experiments/*.json` 不算留存，因為清 build artifacts 就會消失。
- 語意變更同步 Model.md。
- solver 連續 3 輪無改善就升級方向；structure 2 輪不達標就停下回報。

</critical_notes>

<paved_path>

## 三類入口

| 觸發 | 動作 |
| --- | --- |
| solver：timeout / gap 大 / 太慢，模型正確 | 只調 `CplexConfig` |
| data：使用者改參數值 | 只改輸入資料 |
| structure：加刪約束、Big-M、infeasible、reformulation | 回 Model.md，確認後重走轉譯 |

## 常用 solver 決策

| 症狀 | 動作 |
| --- | --- |
| timeout | 提 `timeLimit`、試 `mipEmphasis = 1`、調 threads |
| gap 大 | 收 `epGap`、試 `mipEmphasis = 2`、再考慮 cuts |
| 記憶體爆 | `treeMemoryLimit` + `nodeFileInd` |
| 可重現 | `parallelMode = 1` + 固定 `randomSeed` + `detTimeLimit` |
| 數值不穩 | `numericalEmphasis = true` |

## AI tuning 閉環與 champion promotion

每輪由 AI 執行下列循環；不是由 production process 自改 source：

1. 從 `Program.cs` 的 `productionBaseline` clone 本輪 baseline 與 variants；一次只改一個旋鈕。
2. 用新的 `<Project>-tuning-r<N>` 名稱執行 `OptExperiment`，避免 append 的歷史 Trial 混入本輪判斷。
3. 讀 `result.Trials` 及 `Experiments/<name>.json`；JSON 的 `Trial.Config` 是可重現設定快照，`Trial.Metrics` 是評分依據。
4. 先做 eligibility gate：錯誤、Infeasible、Unbounded、沒有可行解、違反目標品質門檻的 candidate 一律淘汰。
5. 對合格 candidate 比較相同 instance / seed / time limit 下的 Status、objective、gap、runtime、node；正式 tuning 用 3–5 seeds 與 hold-out instances，runtime 以 shifted geometric mean、timeout 以 PAR10 彙總。第一個 solve MUST 當 warm-up 排除，variant 執行順序要跨 seed 輪替或隨機化，避免 cold-start 與固定順序偏差。改善 MUST 大於 baseline 自身 variability，NEVER 用單次牆鐘快幾毫秒就 promotion。
6. champion 勝出後，AI 把其 `ConfigSnapshot.SolverSpecific` 對應值明確寫回 `productionBaseline` initializer；同時更新 initializer 上方 provenance（experiment + Trial label）並先在 `TuningHistory.md` 記錄 before/after config diff。不得在 runtime 寫 source，也不得讓 prod 臨時讀「目前最快的一列」。
7. `dotnet build` 後跑無參數 production；MUST 通過 `OnSolved` / `ValidateRules`，並核對 Status、objective、gap 與輸出，再把驗證結果補回同一筆 history。失敗或回退就撤銷本次 promotion，保留原 baseline並記錄 rejected。
8. 成功後新設定即成為下一輪 baseline。即使沒有 candidate 勝出，也要在 `TuningHistory.md` 記錄 retain 與證據，避免下一輪重做同一組實驗。連續 3 輪無實質改善則停止 solver tuning 或依規範升級方向。

評分採 lexicographic gate，不把不同品質的解只用 runtime 混排：

1. 正確且可接受的 Status／解品質。
2. 達成專案要求的 objective 與 MipGap。
3. 前兩者相同或都達標時，才比較穩健彙總後的 runtime；node / iteration 作診斷與 tie-break。

完成條件不是「產生了 experiment CSV」，而是 production baseline 已 promotion 且 production 驗證通過；若沒有 candidate 能可靠打敗 baseline，保留 baseline 也是合法結論，但 MUST 留下實驗證據。

### TuningHistory.md 固定欄位

每輪一節，至少包含：日期、round、experiment name、baseline Trial、champion／candidate Trial、instances / seeds / 彙總方法、Status / objective / gap、before/after config diff、promotion／retain／rejected 決策、理由、production build 與 `ValidateRules` 結果。這是決策索引；完整逐 Trial 數值仍由 experiment JSON 保存。

</paved_path>

<patterns>

## OptExperiment

`Program.cs` 已載入一份 `data` 並直接定義 `model`；所有 cell 共用它：

```csharp
var productionBaseline = new CplexConfig
{
    epGap = 0.03,
    timeLimit = 300,
    workThreads = 8,
};
var baseline = productionBaseline.Clone();
var emphasis = baseline.Clone();
emphasis.Emphasis = 2;
var tighterGap = baseline.Clone();
tighterGap.epGap = 0.01;

var result = new OptExperiment("<project>-tuning-r1", "一次只改一個旋鈕")
    .AddModel(model)
    .AddConfig("r1-baseline", baseline)
    .AddConfig("r1-emphasis=optimal", emphasis)
    .AddConfig("r1-gap=0.01", tighterGap)
    .Run();
```

- experiment 預設 solver log、LP/MPS/Sol export、housekeeping 都 OFF。
- variant 用 `Clone()` 產具體物件，NEVER 用 tune delegate 突變共用 config。
- `.AddModel` × `.AddConfig` 自動笛卡兒積；單格用 `.AddTrial(model, label, config)`。
- Trial label 為 `ModelName | config-label`；輪次寫進 experiment name 或 label。
- `OnSolved` 只屬 `OptProject`，experiment 不寫大量 solution。
- 載入後把 data 視為唯讀；框架只保護受控 mutation API，直接 public field / mutable list 寫入不保證立即攔截。

## Infeasible

先跑 IIS。只有使用者明確同意時才建立具名 soft model variant；canonical hard model 保持不動。penalty 來自 Parameter，並記錄放鬆限制式與違反量。

</patterns>

<fatal_implications>

- NEVER 未過正確性 gate 就調效能。
- NEVER 以 tuning 名義移項、改號、翻方向或四捨五入。
- NEVER 語意變更不更新 Model.md。
- NEVER 沒有 Experiment 記錄就宣稱改善。

</fatal_implications>
