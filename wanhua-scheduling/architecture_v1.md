# 排班系統 架構文件 v1

> 萬華世界 · 排班系統 · v1 草案 ｜ 2026-04-17
> 本文件以開發者為對象的精簡版。🟡 = 待 Winnie 確認。

---

## 1. 系統目標

取代目前 Google Sheet 排班流程：
- 排班省時（拖拉 + 即時違規檢查）
- 員工查班方便（PWA）
- 減少排班錯誤
- 計算工時（不算薪資）

---

## 2. 使用者角色

| 角色 | 成員 | 權限 |
|---|---|---|
| 高階主管 | Winnie, Fish, 政道 | Winnie 排班；Fish/政道 🟡 僅查看？ |
| 正職 | 超威、蛋頭、思坦 | 填休假表、查班、提換班 |
| PT | 人力 pool 🟡 | 填意願表、查班 |

**登入**：Google Login。

---

## 3. MVP 分階段

### Phase 1（1–2 個月）核心
- Google Login
- 主管建立月度行事曆（國假）
- 正職填月休假表
- PT 填月排班意願表
- 自動計算正職**排班目標工時**
- 主管手動拖拉排班 + 即時違規檢查
- 員工 PWA 查班
- 班表發布 + 個人班表分享連結

### Phase 2（+1 個月）
- 換班申請流程
- 變更 log
- 月報 / 統計
- 打卡比對（🟡 確認有無打卡系統）

### Phase 3（+2 個月）
- 自動排班建議（OR-Tools CP-SAT）
- Excel / PDF 匯出
- 補時數自動追蹤

---

## 4. 核心業務邏輯

### 4.1 月排班週期

```
月底前 N 天：主管公告下月行事曆
  ↓
正職填休假表 / PT 填意願表
  ↓
系統計算「排班目標工時」
  ↓
主管手動排班 + 違規檢查
  ↓
發布班表（月底最後一週）
  ↓
月中異動 → 變更 / 換班流程
```

### 4.2 目標工時（兩個含義）

**重要**：系統核心使用的是「**排班目標工時**」。

```
月工時總額   = (當月總日數 − 固定週休 − 特殊協議 − 國假) × 8H
排班目標工時 = 月工時總額 − 特休時數 − 會議時數    ← 系統用這個
達成判定    = 實排時數 ≥ 排班目標工時
```

🟡 **待 Winnie 確認**：以 4 月超威（152H 排班目標 / 134H 實排 / 16H 特休 / 2H 開會）反推，公式中「月工時總額」是否已經扣除特休天數。推算 168H vs. 實際 152H 差 16H 剛好是特休時數。

### 4.3 硬性限制（必須遵守）

- 工時上限：PT ≤ 90h、不固定 PT ≤ 36h 🟡、主管 ≤ 40h
- 兩班間休息 ≥ 11 小時（勞基法）
- 連續工作：≤ 6 天；純 8h 班 ≤ 5 天；含 10h 班 ≤ 2 天
- 關鍵時段最低正職人力（依日別）
- 同日休假正職人數 ≤ N−1（🟡 N 值待定，原規則 ≤ 3 因只剩 3 位正職失效）
- 同日只排一個班次（禁止 split shift）

### 4.4 軟性限制（應該達成）

- 正職達排班目標工時
- 固定 PT 56–90h（🟡 確認是否還適用）
- 公平性：晚班、長班日次數分布均勻

### 4.5 班表變更 / 換班

- 只有 Winnie 能改班表
- 每次變更留 log
- 員工換班：員工 A 申請 → B 同意 → Winnie 核准

---

## 5. 資料模型（Phase 1）

```typescript
// 員工
employees {
  id: uuid
  name: text
  email: text          // Google Login
  role: 'admin' | 'fulltime' | 'pt'
  active: boolean
  joined_at: date
}

// 正職專用設定
employee_settings {
  employee_id: uuid (FK)
  fixed_rest_days: int[]    // 0=Sun, 1=Mon, ...
  can_work_10h: boolean
  annual_leave_quota: int
  annual_leave_expires_at: date
}

// 班次主檔
shifts {
  code: text PK         // M8, M10, A8, S2, N7.5
  name: text
  start_time: time
  end_time: time
  nominal_hours: numeric
  actual_hours: numeric
  day_type: 'weekday' | 'long_day' | 'sunday' | 'all'
}

// 月度行事曆
monthly_calendars {
  year_month: text PK   // '2026-04'
  holidays: date[]
  makeup_days: date[]
  published: boolean
}

// 正職月休假表
leave_requests {
  id: uuid
  employee_id: uuid (FK)
  year_month: text
  extra_leave_dates: date[]     // 額外休假（非固定週休）
  annual_leave_dates: date[]    // 特休日期
  annual_leave_hours: int       // 特休時數（通常 天 × 8）
  special_agreement_days: int   // 特殊協議（如超威 +1）
  submitted_at: timestamp
  UNIQUE(employee_id, year_month)
}

// PT 月排班意願表
pt_availability {
  id: uuid
  employee_id: uuid (FK)
  year_month: text
  available_slots: jsonb        // [{ date, shift_codes[] }]
  notes: text
  submitted_at: timestamp
  UNIQUE(employee_id, year_month)
}

// 正職月度目標（系統計算）
monthly_targets {
  employee_id: uuid (FK)
  year_month: text
  work_days: int
  total_monthly_hours: int      // 月工時總額
  meeting_hours: int            // 由主管輸入
  schedule_target_hours: int    // 排班目標工時（關鍵）
  carryover_hours: int          // 上月欠 / 超
  computed_at: timestamp
  PRIMARY KEY (employee_id, year_month)
}

// 班表
schedules {
  id: uuid
  date: date
  employee_id: uuid (FK)
  shift_code: text (FK)
  published: boolean
  is_important: boolean
  notes: text
  created_at: timestamp
  created_by: uuid
  INDEX (date, employee_id)
}

// 變更 log
schedule_changes {
  id: uuid
  schedule_id: uuid (FK)
  changed_by: uuid (FK)
  changed_at: timestamp
  action: 'create' | 'update' | 'delete'
  before_json: jsonb
  after_json: jsonb
  reason: text
}

// 換班申請
swap_requests {
  id: uuid
  from_employee_id: uuid
  to_employee_id: uuid
  from_schedule_id: uuid
  to_schedule_id: uuid
  status: 'pending' | 'peer_approved' | 'admin_approved' | 'rejected'
  approved_by: uuid
  created_at: timestamp
}
```

---

## 6. 技術架構

| 層級 | 技術 |
|---|---|
| 前端 | Next.js 15 App Router + TypeScript |
| UI | Tailwind CSS + shadcn/ui |
| 拖拉 | @dnd-kit/core |
| PWA | next-pwa |
| 資料庫 | Supabase (Postgres) |
| 登入 | Supabase Auth + Google OAuth |
| 即時 | Supabase Realtime |
| 部署 | Vercel |
| 排班演算法 (P3) | Google OR-Tools (Python) |

### 專案結構

```
wanhua-scheduling/
├── app/
│   ├── (auth)/login/
│   ├── (admin)/
│   │   ├── calendar/          # 行事曆管理
│   │   ├── leave-requests/    # 檢視休假表
│   │   ├── schedule/          # 排班拖拉介面
│   │   └── employees/         # 員工管理
│   ├── (staff)/
│   │   ├── my/                # 個人班表
│   │   ├── leave/             # 填休假表
│   │   └── schedule/          # 查全店班
│   └── api/
├── components/
│   ├── schedule-grid/         # 拖拉排班主元件
│   ├── constraint-checker/    # 違規檢查
│   └── ui/                    # shadcn
├── lib/
│   ├── supabase/              # client/server/types
│   ├── rules/                 # 規則引擎
│   └── calculations/          # 工時計算
└── supabase/
    └── migrations/            # schema
```

---

## 7. 非功能需求

- **資料保存**：無限（Supabase Postgres）
- **隱私**：員工可看全店班表，不看薪資 / 請假理由
- **效能**：班表查詢 < 500ms
- **備份**：Supabase 自動日備份
- **裝置**：主管桌機、員工手機 PWA
- **瀏覽器**：Safari (iOS)、Chrome (Android/桌機)

---

## 8. 待 Winnie 確認（🟡）

見獨立的 [Winnie_待確認清單.html](Winnie_待確認清單.html)。

---

## 9. 下一步

1. 把架構文件 + 確認清單傳給 Winnie
2. Winnie 回覆後更新 v2
3. 平行動作：
   - 建立 Supabase 專案 + schema migrations
   - 建立 Next.js 專案、部署 Vercel（空殼）
   - 實作 Google Login + employees CRUD

**估計時程**：Phase 1 約 6–8 週能上線，前 2 週骨架不依賴 Winnie 回覆。
