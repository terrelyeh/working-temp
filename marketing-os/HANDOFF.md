# marketing-os — 交接說明（HANDOFF）

給接手開發者（或新的 Claude Code session）：這份說明本 repo 目前的狀態、
設計決策與待辦，讓你不需回看原始對話就能接著做。

## 這是什麼
行銷部門用的 Claude 外掛「市集 repo」。同仁裝一次即可用自然語言觸發行銷技能，
並用一鍵指令鋪好標準工作資料夾（記憶 + 分區隔離）。

## 結構
```
.claude-plugin/marketplace.json     市集目錄（Add from a repository 用；須在 repo 根層）
plugins/marketing-os/
├── .claude-plugin/plugin.json      外掛清單
├── .mcp.json                       佈線 Firecrawl MCP（憑證用 env，不打包）
├── commands/begin-marketing.md     一鍵初始化：輕量 onboarding → 建四個標準夾
└── skills/
    ├── campaign-brief/             活動企劃書
    ├── social-post/                多平台社群貼文
    ├── ad-copy/                    廣告文案 + A/B
    ├── competitor-scan/            競品分析（優先用 Firecrawl 抓真實頁）
    ├── weekly-report/              行銷週報
    └── subfolders/                 隨需建立單一資料夾（帶標準結構）
```

## 關鍵設計決策（改之前先讀）
1. **技能是全域的**：裝好後所有資料夾都能觸發，靠 SKILL.md 的 `description` 觸發語，
   不綁特定資料夾。資料夾只負責「記憶 / 身分 / 指令」的隔離。
2. **狀態外置**：記憶存在使用者工作區的 CLAUDE.md / MEMORY.md，不存在外掛裡。
3. **記憶只由使用者觸發、刪前確認、衝突不覆蓋**（沿用 cowork-os 的克制原則）。
4. **onboarding 去劇場化**：begin-marketing 內含 3~4 題選項式問答，分「團隊層(品牌/合規)」
   與「個人層(通路/風格)」，答案直接寫進 CLAUDE.md，不要求貼設定頁；不幫 AI 取名。
5. **連接器 = 佈線(可打包) + 授權(各自手動)**：.mcp.json 只佈線，FIRECRAWL_API_KEY 等
   憑證一律 env 帶入，絕不寫死打包。桌面/網頁無 shell，.mcp.json 僅 Claude Code 適用。

## 兩種建資料夾方式
- `/marketing-os:begin-marketing` — 開機跑一次，建固定四個標準夾。
- 對話說「建一個 XX 資料夾」 — subfolders 技能隨需建單一夾。

## 如何新增技能（維護）
在 `plugins/marketing-os/skills/` 新增 `<名>/SKILL.md`（最上方 description 就是觸發器）→
`claude plugin validate ./plugins/marketing-os` → plugin.json version +1 → commit/push。
技能不需登記進 manifest。詳見 README.md「維護指南」。

## 待辦 / 可能的下一步
- [ ] 品牌與合規規範：填進 begin-marketing 產生的根 CLAUDE.md「品牌與合規」段。
- [ ] 觸發語體檢：技能變多時，確認各 SKILL.md 的 description 觸發語不互相打架。
- [ ] 視需要加更多連接器（HubSpot/Drive/Canva）；作法同 Firecrawl，憑證走 env。
- [ ] （選）針對真實團隊最常做的 3 件事，客製或新增對應技能。
- [ ] 實跑驗收：跑一次 begin-marketing 看 onboarding 體驗，再微調題目。

## 發佈
本資料夾即 repo 根。git init → 推成獨立 private repo →
同仁「Add from a repository」貼 repo 網址安裝；各自設 FIRECRAWL_API_KEY。
