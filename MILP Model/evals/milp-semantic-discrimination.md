---
id: milp-semantic-discrimination
domain: MILP Model
rule_refs: ["語義判別鐵律", "單位推理守則", "Terminology Mapping Table 持久化在 Model.md", "宣告先於使用"]
created: 2026-07-05
updated: 2026-08-02
---

# Eval Case — milp-semantic-discrimination

## Input（模擬使用者輸入）

幫我把這題建成數學模型：小明的咖啡店（店名叫「晨光」）每天用阿拉比卡豆和羅布斯塔豆調配招牌咖啡。阿拉比卡豆每公斤成本 300 元、羅布斯塔豆每公斤 180 元。要決定每種豆子各用幾公斤。調配後的咖啡因濃度不得超過 1.2%，阿拉比卡咖啡因 1.5%、羅布斯塔 2.2%。每天總共要調出至少 50 公斤咖啡。員工每班 8 小時、時薪 150 元。目標是讓每天豆子成本最低。

## Expected（可觀察判準，全過才 PASS）

- [ ] 產出 Terminology Mapping Table（或等價逐句歸類），每個子句都標了 role（parameter / variable / derived / constraint / objective / irrelevant）
- [ ] Terminology Mapping Table 直接內嵌在唯一的 `<Project>_Model.md`，沒有另外產出或要求建立 `Glossary.md`
- [ ] 「決定每種豆子各用幾公斤」被歸為 **decision variable**（非 parameter）
- [ ] 「咖啡因濃度不得超過 1.2%」這條**隱藏約束**被抓出來（歸為 constraint），未漏
- [ ] 店名「晨光」/ 店主「小明」被標為 **irrelevant**，未靜默略過
- [ ] 成本 300 / 180、濃度 1.5% / 2.2%、50 公斤等數值歸為 **parameter**
- [ ] 未把「時薪 150 × 每班 8 小時」自動推導成每班工資（除非明確標為 derived 並註來源）

## Anti-patterns（出現任一即 FAIL）

- 直接跳到 SET/PARAM/VAR 符號，沒有先做逐句語義判別
- 漏掉咖啡因濃度上限那句（沒歸類、模型裡也沒有這條約束）
- 把「用幾公斤」當成已知 parameter，或把成本 300 當成 variable
- 自動推導 derived 值（每班工資 = 1200）卻沒標 derived / 沒註推導來源
- 把術語表拆成獨立 `Glossary.md`，或要求同時維護 Model.md 與 Glossary.md
- 在本階段就產出任何 `.cs`
</content>
