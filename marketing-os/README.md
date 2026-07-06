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
