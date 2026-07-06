# marketing-os-marketplace

行銷部門的 Claude 外掛市集。內含 `marketing-os` 外掛。

## 目錄結構
```
marketing-os/
├── .claude-plugin/marketplace.json     # 市集目錄（列出有哪些外掛）
└── plugins/
    └── marketing-os/                   # 外掛本體
        ├── .claude-plugin/plugin.json
        ├── commands/begin-marketing.md
        └── skills/{campaign-brief,social-post,ad-copy,competitor-scan,weekly-report}/SKILL.md
```

## 給同仁：怎麼安裝

**方式 A — 從 repository 安裝（推薦，可自動更新）**
1. 把這個 `marketing-os/` 資料夾發佈成一個 GitHub repo（見下）。
2. Claude 桌面/網頁 → Customize → Plugins → **Add → Add marketplace → Add from a repository**，貼上 repo 網址。
3. 在清單裡找到 `marketing-os`，按 Install。
（CLI 等效：`/plugin marketplace add <你的-repo>` 然後 `/plugin install marketing-os@marketing-os-marketplace`）

**方式 B — 直接上傳（最快，無自動更新）**
把 `plugins/marketing-os/` 這個資料夾壓成 zip → Customize → Plugins → **Add → Upload plugin** 拖進去。

## 給你（維護者）：怎麼發佈成 repo
```bash
# 把 marketing-os 這層做成獨立 repo 的根目錄
cd marketing-os
git init && git add . && git commit -m "init marketing-os marketplace"
# 在 GitHub 建一個空 repo 後：
git remote add origin https://github.com/<你>/marketing-os.git
git push -u origin main
```
之後你更新技能 → push → 同仁在 Plugins 介面按更新即可同步。

## 客製
- 改資料夾類別：編輯 `commands/begin-marketing.md` 的第 3 步表格。
- 改技能觸發語：編輯各 `SKILL.md` 最上方的 `description`（那就是觸發器）。
- 加品牌規範：填進 `begin-marketing` 產生的根 `CLAUDE.md`「品牌與合規」段落。

## 維護指南：如何持續新增技能

這個 repo 是一個「活的」技能倉庫，可以一直長大。加東西後 push，同仁在 Plugins 介面按「更新」即可同步——**一次維護、全員受惠**。

### A. 在既有 plugin 裡加一個新技能（最常見）
```bash
# 1. 新增技能資料夾
mkdir -p plugins/marketing-os/skills/<新技能名>

# 2. 建立 SKILL.md，最上方 frontmatter 的 description 就是「觸發器」
cat > plugins/marketing-os/skills/<新技能名>/SKILL.md <<'SKILL'
---
name: <新技能名>
description: >-
  <一句話說明它做什麼>。當使用者要<情境>時使用。觸發說法包括：
  「<觸發語1>」「<觸發語2>」…，或任何類似變體。
---

# <技能標題>
## 流程
1. ...
## 規範
- 遵守資料夾 CLAUDE.md 的品牌 tone 與合規；數據不編造。
SKILL

# 3. 驗證結構
claude plugin validate ./plugins/marketing-os

# 4. 版本號 +1（plugins/marketing-os/.claude-plugin/plugin.json 的 version）
#    例如 1.0.0 -> 1.1.0

# 5. 提交推送
git add . && git commit -m "feat(skill): 新增 <新技能名>" && git push
```
> **技能不需登記進 manifest**：放進 `skills/` 就會被自動掃描，`plugin.json` 不用改（除了建議把 version 往上加）。

### B. 加一個「整包新 plugin」（例如日後的 sales-os）
1. 新增 `plugins/<新plugin>/`，內含 `.claude-plugin/plugin.json` 與 `skills/`、`commands/`。
2. **編輯 `.claude-plugin/marketplace.json`**，在 `plugins` 陣列多列一筆（`name` + `source` 指向該資料夾）。
3. 驗證、加版本、push。同仁的市集清單就會多出這個可裝的 plugin。

### 版本與變更慣例
- 每次有意義的變更就把 `version` 往上加：修 bug → patch（1.0.**1**）；加技能 → minor（1.**1**.0）；破壞相容 → major（**2**.0.0）。
- commit 訊息用 `feat(skill): …` / `fix(skill): …` 標明改了什麼，方便日後追。

### 加技能時的注意事項
- **觸發語別打架**：技能一多，`description` 的觸發語容易同時命中多個技能。新技能的觸發語要具體、與現有的不重疊。
- **產出格式固定**：延續現有技能「固定輸出區塊 + 規範段」的寫法，全團隊產出才一致。
- **先驗證再 push**：養成 `claude plugin validate` 的習慣，避免把壞掉的結構推給全員。
