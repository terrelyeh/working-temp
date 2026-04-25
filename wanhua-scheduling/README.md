# 萬華世界排班系統

為**萬華世界下午酒場**（台北餐酒館）開發的排班自動化系統。
透過 Claude Skill 與規則引擎，取代手工跑 Google Form + Google Sheet 的流程，
讓主管每月排班從 6–8 小時降到 1–2 小時。

---

## 為什麼做這個

排班的痛點集中在「重複的黑工」：

- 月底主管手動建 Google Form、改日期、改活動註記、寫每位正職的預計天數
- 員工填表常誤勾、漏勾、姓名格式不一（91 ↔ 91.0）
- 對帳年假餘額、算目標工時都靠人眼
- 純手動拖拉 Google Sheet 排班，2–3 天才完成
- 月中換班、臨時請假沒紀錄、容易漏通知

## 目前進度

### ✅ 已完成

- **規則文件 v3** — 17 章節完整規則書，已整合 Winnie + Fish + 政道確認
- **休假表驗證 skill**（`wanhua-leave-validate`）— 自動檢查月休假表 / PT 意願表，產出報告 + 清理 JSON 給排班用
- **排班 UI demo** — 員工版、全店時間軸（Gantt）、月度統計、搭班查詢
- **Winnie Q&A 完整紀錄** — 第一輪 24 題已全數回覆，作為業務邏輯詳細參考

### 🔜 開發中 / Pending

- 月初 form 前置 skill（`wanhua-monthly-prep`）
- **自動排班 solver**（`wanhua-schedule`）— 用 Google OR-Tools CP-SAT
- 班表發布 skill（`wanhua-publish`）— LINE 通知 + 個人班表

### 📅 未來 Phase

- **Phase 2** PWA 填表（取代 Google Form），即時驗證、年假餘額顯示
- **Phase 3** 全平台（員工 PWA + 主管後台 + iChef 打卡資料整合）

## 技術架構

| 層級 | 選擇 |
|---|---|
| Skill 語言 | Python 3.12 + uv |
| Solver | Google OR-Tools CP-SAT（Phase 1 已選定） |
| 資料 | YAML（規則 / 設定）+ Excel（form 匯出）+ JSON（中介） |
| 文件 | HTML + Markdown，部署 GitHub Pages |
| 未來前端 | Next.js + Vercel |
| 未來後端 | Supabase（DB + Auth）+ Python solver service |
| Solver 部署 | Railway / Fly.io / Cloud Run（Vercel 不適合長時計算） |

## 線上文件

入口：**https://terrelyeh.github.io/working-temp/wanhua-scheduling/**

| 文件 | 對象 |
|---|---|
| [📜 排班規則 v3](https://terrelyeh.github.io/working-temp/wanhua-scheduling/rules-v3.html) | 全員 — 業務規則權威 |
| [🏗️ 流程設計文件](https://terrelyeh.github.io/working-temp/wanhua-scheduling/workflow-design.html) | 開發者 — 路線圖 |
| [🟡 臨時缺席確認題](https://terrelyeh.github.io/working-temp/wanhua-scheduling/ad-hoc-absence-questions.html) | Winnie — 待回覆 |
| [📋 Winnie 第一輪 Q&A](https://terrelyeh.github.io/working-temp/wanhua-scheduling/winnie-checklist.html) | 詳細業務邏輯參考 |
| [🗓️ UI Demo](https://terrelyeh.github.io/working-temp/wanhua-scheduling/demo/) | 概念展示 |

## 快速開始（開發者）

```bash
# 進入第一支 skill 資料夾
cd skill/wanhua-leave-validate

# 安裝相依（uv 自動建 venv）
uv sync

# 執行月驗證（範例：2026 年 5 月）
uv run python validate.py 2026-05

# 輸出：
#   report/2026-05-validation-report.md   ← 給 Winnie 看的驗證報告
#   report/2026-05-clean-leave.json       ← 清理後資料，餵下一支排班 skill
```

## 專案目錄

```
萬華排班/
├── docs/                       文件源（HTML / MD）
├── skill/wanhua-leave-validate/  休假表驗證 skill（已完成）
│   ├── config/                 員工名冊 + 規則
│   └── data/YYYY-MM/           每月輸入資料
├── demo/                       UI demo
└── .claude/skills/             Claude Skill 註冊
```

## 給新進開發者

新 Claude session 接手時請先讀 `CLAUDE.md`（AI 工作備忘錄）
或 `docs/workflow_design.md`（人類版完整路線圖）取得 context。

- **詳細業務規則** → [docs/排班規則_v3.html](docs/排班規則_v3.html)
- **規則的設計理由** → [Winnie_待確認清單.html](Winnie_待確認清單.html)
- **下一步該做什麼** → CLAUDE.md 的 Next Steps

## 授權與使用

內部專案。萬華世界下午酒場專用。
