# 萬華世界排班流程優化設計文件

> 目的：記錄目前排班流程的痛點、未來流程的分階段設計、以及流程層面的優化建議。
> 後續新 session 實作時可直接餵此文件當 context。
>
> 最後更新：2026-04-24
> 相關文件：
> - 規則：[`docs/排班規則_v3.html`](./排班規則_v3.html)、[`skill/wanhua-leave-validate/config/rules.yaml`](../skill/wanhua-leave-validate/config/rules.yaml)
> - 架構：[`architecture_v1.md`](./architecture_v1.md)（早期版本，可能已部分過時）
> - 實作：[`skill/wanhua-leave-validate/`](../skill/wanhua-leave-validate/)

---

## 1. 現況痛點

目前每月排班流程是 Winnie 手工跑 Google Form + Google Sheet，主要痛點：

| 階段 | 痛點 | 代價 |
|---|---|---|
| 月底建 form | Winnie 手動改日期、國假標註、活動備註、每位正職的預計天數說明 | 30–60 分鐘/月 重工 |
| 員工填表 | 誤勾、忘勾、姓名格式不一（如 91 ↔ 91.0）、重複提交 | 驗證耗時、LINE 來回溝通 |
| 彙整資料 | Form CSV 匯出 → 人工對帳年假餘額 → 人工算目標工時 | 易算錯 |
| 排班 | 純手動在 Google Sheet 拖班次、試組合 | 2–3 天、常晚上排 |
| 發班表 | 貼 Google Sheet 連結到 LINE 群組 | 員工查班痛苦（試算表複雜） |
| 月中異動 | 換班、臨時請假靠 LINE 協調、Winnie 手動改 Sheet | 沒紀錄、容易漏通知 |
| 月底對帳 | 打卡 vs 排班人眼比對；年假餘額塞備註欄 | 欠時數沒追蹤、易漏 |

## 2. 未來流程分三階段

每階段可獨立交付，逐步取代現有流程。

### Phase 1 — Skill 層自動化（最短 2 週）

**定位**：員工端仍用 Google Form + LINE，Winnie 端用 Claude Skill 自動化所有「黑工」。
改動最小、風險低、見效快。

```
月 19 號 ─────────────────────────
  Winnie 執行：/wanhua-monthly-prep 2026-06
  → Skill 自動：
      ① 讀 Google Calendar → 產出下月活動清單
      ② 算每位正職當月預計目標工時（含特休餘額）
      ③ 生出 form 題目文字（給 Winnie 複製到 Google Form）

月 20 號 ─────────────────────────
  Winnie 貼到 Form、發出

月 25 號 ─────────────────────────
  Form 截止
  Winnie 執行：/wanhua-leave-validate 2026-06
  → Skill 驗證、列缺交、標異常 → 產出 markdown 報告 + clean JSON

月 27 號 ─────────────────────────
  Winnie 執行：/wanhua-schedule 2026-06
  → Skill 跑 OR-Tools constraint solver 排班 → 產出 1–3 個可行方案
  → Winnie 在 UI 挑一個、微調

月 28 號 ─────────────────────────
  Winnie 執行：/wanhua-publish 2026-06
  → 產員工個人班表（PNG/URL）
  → 一鍵貼 LINE 群組
  → 年假結算、補時數記錄寫回
```

**預計 Winnie 工時投入：6–8h → 1–2h**

**需要開發的 Skill**：
- `/wanhua-monthly-prep` — 月初前置
- `/wanhua-leave-validate` ✅ 已完成
- `/wanhua-schedule` — solver
- `/wanhua-publish` — 發布 + 結算

### Phase 2 — 填表自助化（+ 2 個月）

**定位**：自己做一個 PWA 表單取代 Google Form，填表體驗大升級。

新增功能：
- **即時檢查**：勾到週一（所有人排休）自動帶入；會議日標灰色不可勾
- **年假餘額即時顯示**：填表時看到「超威剩餘 17 天」
- **衝突偵測**：3 位正職同日全休 → 跳紅字「需 Winnie 確認」
- **填完自動進 DB**：不用再 export CSV
- **手機 PWA**：員工加到主畫面、填表體驗好

後端：Supabase + Next.js（對應早期架構文件）。

### Phase 3 — 全平台（+ 2 個月）

**定位**：取代 LINE 群組溝通 + Google Sheet 查班，完整系統。

- **員工 PWA**：個人班表、換班申請、查班
- **主管後台**：排班拖拉、違規即時提示、月報
- **自動對帳**：打卡系統（iChef）匯出 × 排班資料比對
- **推播**：LINE Notify 整合

## 3. 流程層優化建議（跟系統無關、立即可用）

### 建議 ① Form 發送前，先寄「當月預設答案」給員工

**問題**：目前員工每月自己勾 13–18 個日期，容易誤勾漏勾。
**做法**：Skill 在 Winnie 發 form 前，先寄個人化預設訊息：

> 🫰 **超威 6 月預設休假**（若維持原本不用動）：
> - 固定休：週一、二 共 8 天
> - 國定假日：0 天
> - 特殊協議：🟡 本月是否有賣漢堡？
> - 預計目標工時：144h
> - 剩餘年假：17 天
>
> 若要調整，只在 form 勾「和預設不同」的部分

**好處**：
- 正職多數月份完全不用動腦（預設就對）
- 只標「變動」項目 → 減少誤勾、Winnie 一眼看出誰有變動
- 搭配「Skill 驗證總天數對帳」邏輯，異常容易發現

### 建議 ② PT / 外援改填「不可排日」不填「可排日」

**問題**：PT 每月要勾 15–20 個可上班日 → 耗時。
**做法**：翻轉語意，改勾 5–8 個「不能上的日子」（其餘預設可排）→ 填表時間少一半。

### 建議 ③ 會議時數自動帶入班表

**問題**：Winnie 手動加「每月第一週二店務例會 2h」到正職班表。
**做法**：Skill 自動判斷當月第一個週二、為每位正職加 2h 會議時數，不用手填。

### 建議 ④ 歷史資料累積後啟用「智慧推薦」

3 個月資料後，Skill 開始學：
- 誰常請特休 → 排班時預留彈性
- 誰常上晚班 → solver 公平性提醒
- 哪些日子常缺班 → 提前通知凱柏

### 建議 ⑤ 活動分級自動處理（已納入 §14 規則 v3）

在 calendar.yaml 每個 event 標 `level`，solver 自動處理：
- `internal` 只標示、`meeting` 只加會議時數、`event_light` 強制指定人員、`event_heavy` 加人力。
詳見 rules-v3.html §14。

## 4. 待 Winnie 決策的開放問題

這些會影響 Phase 1 的細節：

- [ ] **PT 填表方向**：「可排日」 vs「不可排日」哪個好？還是維持目前？
- [ ] **正職「預設答案」形式**：要嗎？LINE 私訊？Email？還是仍希望手動填？
- [ ] **班表公告格式**：LINE 卡片？個人 PNG？月曆 URL？
- [ ] **月中異動**：員工自助換班？還是繼續 Winnie 單一入口？
- [ ] **活動臨時加人**：post-schedule 手動加可以，re-optimize 其他天要不要自動觸發？

## 5. 實作優先順序建議

依「見效快 × 風險低」排序：

1. **`wanhua-leave-validate`** ✅ 已完成
2. **`wanhua-monthly-prep`**（月初 form 前置） — 低風險、省 Winnie 30–60 分鐘/月
3. **`wanhua-schedule`** solver 核心 — 最高價值但複雜；建議先做「規則驗證 + 半自動排班」版本，不要一次做滿
4. **`wanhua-publish`** — 產個人班表 + LINE 卡片
5. **Phase 2 PWA 填表** — Phase 1 運作 2–3 個月後再評估
6. **Phase 3 全平台** — 依實際需要

## 6. 技術決策

- **Solver 選型**：Google OR-Tools CP-SAT（Python）
- **Skill 語言**：Python + `uv`
- **部署**：本機跑 skill（無伺服器成本）
- **未來 web**：Next.js on Vercel + Supabase（Phase 2 時決定）
- **Solver 執行環境**：不適合 Vercel（時間/記憶體限制），未來若上 web 可考慮 Railway / Fly.io / Google Cloud Run 跑 Python 服務

## 7. 關鍵資料流

```
┌─────────────────────────────────────────────────┐
│                  月排班 pipeline                  │
└─────────────────────────────────────────────────┘

[config/employees.yaml]        員工名冊（姓名/角色/固定休）
[config/rules.yaml]            規則權威來源
           │
           ▼
[data/YYYY-MM/calendar.yaml]   月行事曆（國假、活動 + level、特殊協議）
[data/YYYY-MM/annual-leave.yaml] 年假餘額（月初快照）
           │
           ▼
      monthly-prep
           │
           ▼  (輸出：form 題目給 Winnie 複製)
    ┌──────────────┐
    │ Google Form  │ ← 正職 + PT + 外援 填
    └──────┬───────┘
           │
           ▼  (匯出 xlsx)
[data/YYYY-MM/fulltime-form.xlsx]
[data/YYYY-MM/pt-form.xlsx]
           │
           ▼
   leave-validate
           │
           ▼
[report/YYYY-MM-validation-report.md]  給 Winnie
[report/YYYY-MM-clean-leave.json]      給 scheduler
           │
           ▼
      schedule (solver)
           │
           ▼
[report/YYYY-MM-schedule.json]         班表
[report/YYYY-MM-violations.md]         軟規則違反記錄
           │
           ▼
       publish
           │
           ▼
  - LINE 卡片訊息
  - 員工個人班表 URL
  - 年假 / 補時數 更新回寫
```

## 8. 相關已交付內容

- ✅ 排班規則 v3（HTML + yaml）：https://terrelyeh.github.io/working-temp/wanhua-scheduling/rules-v3.html
- ✅ `wanhua-leave-validate` skill + 5 月驗證報告
- ✅ UI demo（員工 / 全店 / 月度統計 / 搭班查詢）：https://terrelyeh.github.io/working-temp/wanhua-scheduling/demo/
- ✅ 給 Winnie 的確認題（臨時缺席）：https://terrelyeh.github.io/working-temp/wanhua-scheduling/ad-hoc-absence-questions.html

## 9. 新 session 繼續的起手式

若要在新 session 實作下一支 skill，建議開場：

```
我要開始實作 wanhua-schedule skill。
請先讀這份 workflow_design.md 了解整體流程，
再讀 rules-v3.html 和 rules.yaml 拿到規則細節，
最後參考 skill/wanhua-leave-validate/ 的架構作為基底。

本 skill 的輸入是 clean-leave.json（由 validate skill 產出），
輸出是 schedule.json。核心是 OR-Tools CP-SAT 排班。
```
