# CLAUDE.md — 萬華世界排班系統

> Last updated: 2026-04-25

## Project Overview

萬華世界下午酒場（台北餐酒館）的排班系統。取代 Winnie 目前手工跑 Google Form + Google Sheet 的流程，
透過 Claude Skill + 未來的 web app 自動化排班。產品定位與功能清單見 [README.md](README.md)。

目前進度：Phase 1（Skill 自動化）的第一支 skill `wanhua-leave-validate` 已完成；
等 Winnie 回覆第二輪確認題後，下一步開始實作 `wanhua-schedule` solver。

## Tech Stack

- **Skill 語言**: Python 3.12 + uv（依使用者全域規則）
- **Solver**（規劃中）: Google OR-Tools CP-SAT
- **資料格式**: YAML（config/規則）+ Excel xlsx（Google Form 匯出）+ JSON（skill 中介輸出）
- **文件**: HTML + Markdown，部署到 GitHub Pages（terrelyeh/working-temp/wanhua-scheduling/）
- **Demo UI**: 純前端（vanilla HTML/CSS/JS，無 build step）
- **未來 web app**: Next.js + Supabase + Vercel；Solver 服務跑在 Railway / Fly.io / Cloud Run

## Directory Structure

```
萬華排班/
├── docs/                            文件源（HTML / MD）
│   ├── index.html                   文件 hub
│   ├── 排班規則_v3.html              17 章節權威規則書
│   ├── workflow_design.{html,md}    流程與架構（給 Claude 讀的是 .md）
│   └── 問題_臨時缺席與補時數.html     第二輪 Winnie 確認題（待回覆）
├── Winnie_待確認清單.html            第一輪 24 題 Q&A 完整紀錄（業務邏輯詳細參考）
├── skill/wanhua-leave-validate/    第一支 skill（已完成）
│   ├── validate.py                  主程式
│   ├── config/employees.yaml       員工名冊（外號 / 角色 / 固定週休 / aliases）
│   ├── config/rules.yaml           規則機器讀版（與 排班規則_v3.html 同步）
│   ├── data/YYYY-MM/               每月輸入（form xlsx + calendar + annual-leave）
│   └── report/                     輸出（validation-report.md + clean-leave.json）
├── demo/index.html                 排班 UI demo（員工版 / 全店時間軸 / 搭班查詢）
└── .claude/skills/wanhua-leave-validate/SKILL.md   Claude Skill 觸發定義
```

## Architecture & Data Flow

月排班 pipeline（每月跑一次）：

```
config/employees.yaml + config/rules.yaml          ← 靜態
+ data/YYYY-MM/calendar.yaml                       ← 月行事曆 + 活動 + 特殊協議
+ data/YYYY-MM/annual-leave.yaml                   ← 年假月初快照
+ data/YYYY-MM/{fulltime,pt}-form.xlsx             ← Google Form 匯出
        ↓
   wanhua-leave-validate（已完成）
   → validation-report.md（給 Winnie）
   → clean-leave.json（給下一支 skill）
        ↓
   wanhua-schedule（TODO，OR-Tools solver）
   → schedule.json + violations.md
        ↓
   wanhua-publish（TODO）
   → LINE 卡片 / 個人班表 URL / 年假與補時數寫回
```

正職目標工時公式：
```
月工時總額 = (當月天數 − 固定週休 − 特殊協議 − 非週一國假) × 8h
排班目標工時 = 月工時總額 − 特休時數 − 會議時數（每月固定 2h）
```

完整資料流圖與 phase 規劃見 [docs/workflow_design.md](docs/workflow_design.md)。

## Conventions

- **規則 yaml ⇄ 規則 HTML 同步**：改 `rules.yaml` 必須同步改 `排班規則_v3.html`（solver 讀 yaml；團隊看 HTML）
- **員工識別用「外號」**（超威、思坦、Sin、91 等），名冊提供 `aliases` 欄位處理 form 上不同寫法
- **月度資料**統一放 `data/YYYY-MM/`，每月一個資料夾
- **HTML 設計**參考 wanhua-dashboard 風格：深色 `#2C3345` navbar + sticky 左側 sidebar TOC + JetBrains Mono code
- **手機 breakpoint** ≤1100px → inline collapsible TOC（不是 drawer，避免疊圖）
- **公開文件都要有 GitHub Pages URL** 方便團隊溝通
- **不寫進公開規則的功能**（員工相容性等敏感設定）放 `workflow_design.md` 附錄 A

## Current Status

功能清單見 [README.md](README.md)。

### 🔜 Next Steps

1. **等 Winnie 回覆**第二輪確認題（共 4 題：病假計入工時、臨時事假 SOP、補時數補不完、長期缺席）— [線上連結](https://terrelyeh.github.io/working-temp/wanhua-scheduling/ad-hoc-absence-questions.html)
2. 實作 `wanhua-monthly-prep` skill — 產 form 題目給 Winnie 複製貼上
3. **實作 `wanhua-schedule` skill** — 核心，用 OR-Tools CP-SAT
4. 實作 `wanhua-publish` skill — 個人班表 + LINE 通知

### 仍 🟡 待 Winnie 拍板的規則

- 正職病假（普通傷病、半薪）是否計入月工時達成（目前 skill 預設「計入」）

## Deployment

```bash
# 跑驗證 skill（範例：2026 年 5 月）
cd skill/wanhua-leave-validate
uv run python validate.py 2026-05
# 輸出：report/2026-05-validation-report.md + report/2026-05-clean-leave.json

# 推送文件到 GitHub Pages（docs 不在本地 git，須手動同步）
cp docs/檔名.html /tmp/working-temp-clone/wanhua-scheduling/
cd /tmp/working-temp-clone
git add -A && git commit -m "..." && git push origin main
# 線上：https://terrelyeh.github.io/working-temp/wanhua-scheduling/
```

## Common Pitfalls

- **本地不是 git repo**：所有公開文件靠手動 cp 到 `/tmp/working-temp-clone`（terrelyeh/working-temp 的 clone）後 push。Skill 程式碼**沒有版本控制**，要小心
- **`/tmp/working-temp-clone` 偶爾被清空**：用 `gh repo clone terrelyeh/working-temp /tmp/working-temp-clone` 重 clone
- **員工姓名格式坑**：Excel 把「91」存成 `91.0`；Form 填「鄭景文」但名冊外號是「景文」。`employees.yaml` 的 `aliases` 處理這些
- **特殊協議逐月確認**：超威「賣漢堡 +1 天休假」不是永久設定。`extra_monthly_days` 預設 0；每月在 `calendar.yaml` 的 `special_agreements` 區塊標記
- **正職國假補償雙層**：不論是否上班都多補 1 天假；若被排上班再加 double pay（≈ 3× 薪資）。PT/外援只有上班才有 double pay
- **同日休假上限不是 hard rule**：Winnie 用人治處理（form 內註明禁休或限 1 位正職）。Solver 不該硬擋
- **PT 給班順序只在國假適用**：「優先給班的人多排 1–2 天」只在國定假日（搶 double pay）；一般日子則是工時平均
- **規則文件 mobile breakpoint 1100px**：寬度小於此會切換成 inline TOC（不是 900px，避免窄視窗 sidebar + content 互相擠壓）
- **排班規則 17 章節**：v3 加入 §14 活動與班表動態調整後，原 §14-16 順延成 §15-17

## 詳細文件

- [docs/排班規則_v3.html](docs/排班規則_v3.html) — 17 章節權威規則書
- [docs/workflow_design.md](docs/workflow_design.md) — 流程與架構設計（**新 session 必讀**；HTML 版同名）
- [Winnie_待確認清單.html](Winnie_待確認清單.html) — 24 題完整 Q&A，業務邏輯詳細出處
- [docs/問題_臨時缺席與補時數.html](docs/問題_臨時缺席與補時數.html) — 第二輪確認題
- [skill/wanhua-leave-validate/README.md](skill/wanhua-leave-validate/README.md) — 第一支 skill 說明
- 線上文件 hub：https://terrelyeh.github.io/working-temp/wanhua-scheduling/
