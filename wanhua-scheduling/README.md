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

### ✅ Phase 1 已完成（演算法驗證）

- **規則文件 v3** — 17 章節完整規則書，已整合 Winnie + Fish + 政道確認
- **休假表驗證 skill**（`wanhua-leave-validate`）— 檢查月休假表 / PT 意願表
- **排班 solver**（`wanhua-schedule`）— OR-Tools CP-SAT，2 秒跑出 OPTIMAL；兩階段求解：Pass 1 完全尊重員工偏好（HARD constraint），INFEASIBLE 才降為 Pass 2 加 penalty
- **5 月對比報告** — 跟 Winnie 手排逐日比對，找出 5 個 disconnect 整理成確認題
- **排班 UI demo** — 員工版、全店時間軸（Gantt）、月度統計、搭班查詢
- **Winnie Q&A 紀錄** — 第一輪 24 題已全數回覆 + 第二輪 4 題 + 第三輪 6 題待回

### 🔜 Phase 2（1 個月內，給團隊用的第一版）

- 規則微調（依 5 月對比結果：S2 拿掉、活動 override 強化、缺人時不強塞）
- 設計 DB schema（Supabase）
- Next.js + Supabase web app（主管後台拖拉排班 + 員工 PWA 查班/填表）
- Python solver service 部署到 Cloud Run
- Google Login + LINE 通知

### 📅 Phase 3 未來

- iChef 打卡資料整合（排班 vs 實際出勤對比）
- 員工自助換班流程
- 進階管理工具（員工相容性軟限制等，見 workflow_design 附錄 A）

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
# 1. 跑 form 驗證（產出 clean-leave.json）
cd skill/wanhua-leave-validate && uv sync && uv run python validate.py 2026-05

# 2. 跑排班 solver（吃 clean-leave.json，產出 schedule.json）
cd ../wanhua-schedule && uv sync && uv run python schedule.py 2026-05

# 輸出：
#   wanhua-leave-validate/report/2026-05-validation-report.md
#   wanhua-leave-validate/report/2026-05-clean-leave.json
#   wanhua-schedule/report/2026-05-schedule.json
#   wanhua-schedule/report/2026-05-violations.md
```

## 專案目錄

```
萬華排班/
├── docs/                              文件源（HTML / MD）
├── skill/
│   ├── wanhua-leave-validate/        ✅ form 驗證
│   └── wanhua-schedule/              ✅ OR-Tools 排班 solver
├── demo/                              UI demo
└── .claude/skills/                    Claude Skill 註冊
```

## 給新進開發者

新 Claude session 接手時請先讀 `CLAUDE.md`（AI 工作備忘錄）
或 `docs/workflow_design.md`（人類版完整路線圖）取得 context。

- **詳細業務規則** → [docs/排班規則_v3.html](docs/排班規則_v3.html)
- **規則的設計理由** → [Winnie_待確認清單.html](Winnie_待確認清單.html)
- **下一步該做什麼** → CLAUDE.md 的 Next Steps

## 授權與使用

內部專案。萬華世界下午酒場專用。
