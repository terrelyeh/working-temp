# Claude 外掛（Plugin）與團隊 OS —— 知識文件

> 整理自實作 `marketing-os` 的討論。目的：讓你（或同事、或新的 Claude session）
> 一次搞懂 Skill / Plugin / Marketplace 的關係、如何打包分享、以及如何用這套機制
> 建立「團隊作業系統」。

## 目錄
1. [核心概念：Skill、Plugin、Marketplace](#1-核心概念)
2. [Plugin 能打包什麼](#2-plugin-能打包什麼)
3. [Plugin 與 Marketplace 的檔案結構](#3-檔案結構)
4. [安裝方式與介面差異（CLI vs 桌面/網頁）](#4-安裝方式)
5. [更新機制：為什麼 marketplace 能自動更新](#5-更新機制)
6. [關鍵心智模型：技能是全域，資料夾是隔離](#6-關鍵心智模型)
7. [團隊 OS 概念：CLAUDE.md / MEMORY.md](#7-團隊-os-概念)
8. [建資料夾：一鍵批次 vs 隨需建立](#8-建資料夾)
9. [Onboarding 設計：團隊層 vs 個人層](#9-onboarding-設計)
10. [接真實數據：連接器 / MCP / CLI](#10-接真實數據)
11. [擴充：加技能(B) vs 加 plugin(C)](#11-擴充)
12. [維護、版本與最佳實務](#12-維護與最佳實務)
13. [安全性](#13-安全性)
14. [決策速查表](#14-決策速查表)
15. [給同事的三句話安裝說明](#15-給同事的三句話安裝說明)

---

## 1. 核心概念

三個名詞，層層包起來：

| 名詞 | 是什麼 | 比喻 |
|---|---|---|
| **Skill（技能）** | 單一能力：一個 `SKILL.md`，寫「遇到某類任務怎麼做」。Claude 需要時才載入 | 一份食譜 |
| **Plugin（外掛）** | 打包單位：把 skills / 指令 / 子代理 / hooks / MCP 包成一份，方便分享、版本控管、自動更新 | 一整盒料理包 |
| **Marketplace（市集）** | 一個 git repo，列出「這裡有哪些 plugin」，負責發現、版本追蹤、更新 | 料理包的架上目錄 |

**重點**：Plugin ≠ Skill。所有 skill 都能放進 plugin，但 plugin 不只是 skill；你也可以只用單獨的 skill 不做成 plugin。Plugin 的價值在「**打包 + 分享 + 一鍵安裝 + 自動更新**」。

### Command vs Skill（改東西前必懂）
| | Command（指令） | Skill（技能） |
|---|---|---|
| 誰觸發 | **人**：手動打 `/指令` | **Claude**：讀對話自動判斷 |
| `description` 作用 | 選單說明文字 | **就是觸發器**——決定會不會被叫到 |
| 適合 | 一次性、儀式性、想主動叫出的流程 | 一句話就該發生的反射動作 |

---

## 2. Plugin 能打包什麼

一個 plugin 可同時含以下任意組合：
- **Skills** — 特定任務的知識與流程
- **Commands（斜線指令）** — `/plugin:command`
- **Subagents（子代理）** — 專責某類工作的代理
- **Hooks** — 生命週期事件（SessionStart、PostToolUse…）自動觸發的行為
- **MCP servers** — 連外部工具/資料（連接器）
- 其他：LSP、output styles、`bin/`（加進 PATH 的執行檔）等

---

## 3. 檔案結構

### Plugin 本體
```
my-plugin/
├── .claude-plugin/
│   └── plugin.json         ← 唯一放這裡的檔案（manifest）
├── commands/xxx.md         ← 斜線指令
├── skills/xxx/SKILL.md     ← 技能（放進來就自動掃描，不需登記進 manifest）
├── agents/xxx.md           ← 子代理
├── hooks/hooks.json
└── .mcp.json               ← MCP 連接器設定
```
**規則**：只有 `plugin.json` 放進 `.claude-plugin/`；其他資料夾一律放 plugin 根目錄。

最小 `plugin.json`：
```json
{ "name": "my-plugin", "description": "...", "version": "1.0.0", "author": { "name": "You" } }
```

### Marketplace（外層）
```
my-marketplace/                       ← 這是 git repo 的「根」
├── .claude-plugin/
│   └── marketplace.json              ← 列出有哪些 plugin（須在 repo 根層）
└── plugins/
    └── my-plugin/                    ← plugin 本體（同上）
```
`marketplace.json`：
```json
{
  "name": "my-marketplace",
  "owner": { "name": "You" },
  "plugins": [
    { "name": "my-plugin", "source": "./plugins/my-plugin", "version": "1.0.0" }
  ]
}
```
`source` 可為相對路徑、GitHub（`{"source":"github","repo":"owner/repo","ref":"v1"}`）、git URL、npm 等。

> ⚠️ **`marketplace.json` 必須在 repo 根層**，否則「Add from a repository」找不到。

---

## 4. 安裝方式

三種方式，**不一定要走 marketplace**：

| 方式 | 做法 | 自動更新 | 適合 |
|---|---|---|---|
| **A. Marketplace（repo）** | 加市集 → install | ✅ 有 | 團隊分享、長期維護 |
| **B. Upload（本機上傳）** | 直接拖 plugin 資料夾/zip | ❌ 無 | 一次性、臨時試 |
| **C. 本機載入（CLI）** | `claude --plugin-dir ./my-plugin` | — | 開發測試 |

### CLI vs 桌面/網頁 介面差異
| | Claude Code CLI | 桌面 App / claude.ai 網頁 |
|---|---|---|
| 管理入口 | `/plugin` 指令 | Customize → Plugins 圖形介面 |
| 加自訂 marketplace | ✅ | ✅ **Add → Add marketplace → Add from a repository** |
| Upload 本機 plugin | ✅ | ✅ **Add → Upload plugin** |
| 打包/跑 CLI 型 MCP（`npx …`） | ✅ | ❌ 無通用 shell |

CLI 指令速記：
```
/plugin marketplace add <owner/repo|git URL|本機路徑>
/plugin install <plugin>@<marketplace>
/plugin update <plugin>@<marketplace>
claude plugin validate .        # 發佈前驗證結構
```

> 注意：**Upload / 自訂 repo 的 plugin 不經 Anthropic 審核**，可帶 hooks/MCP/執行檔，
> 等於會在安裝者機器上跑東西——只在信任來源時使用。

---

## 5. 更新機制

**marketplace 與 upload 最核心的差別 = 有沒有一條「連回來源」的線。**

- **Marketplace（repo）**：記住 plugin 來自哪個 git repo → 你 push 新版，同事按「更新」就同步。**一次維護、全員受惠**，且有版本比對。
- **Upload**：只是把當下檔案複製進去，裝完就斷線 → 要更新只能重新上傳、逐一重裝。

→ 團隊長期用，**一定選 marketplace**。

---

## 6. 關鍵心智模型

> **技能 = 裝一次就到處能用的工具箱；資料夾 = 各個工作台，決定在這張台子上拿哪些工具、用什麼身分、記得哪些事。**

- **技能是「全域」的**：plugin 一旦安裝，所有資料夾都能觸發該技能（靠 `description` 觸發語），**不綁特定資料夾**。你無法把某技能「鎖」在某資料夾。
- **資料夾負責「隔離」**：每個資料夾的 `CLAUDE.md`（指令/身分）與 `MEMORY.md`（記憶）彼此獨立，不互相污染。

這對團隊是好事：你本來就希望大家在哪都能叫出技能，而資料夾負責把各專案的狀態收好。

---

## 7. 團隊 OS 概念

把 plugin 當「引擎」，把使用者資料夾裡的兩個檔當「存檔」：

| 檔案 | 角色 |
|---|---|
| `CLAUDE.md` | 該資料夾的**常駐指令**（每次進來自動讀） |
| `MEMORY.md` | 該資料夾的**持久記憶**（跨 session 筆記） |

核心設計原則（成熟、安全導向）：
1. **狀態外置**：記憶放使用者資料夾，不放 plugin 裡 → plugin 可更新替換，資料留在使用者手上。
2. **記憶只由使用者觸發**：Claude 絕不自動寫入 `MEMORY.md`，只有使用者說「記一下 / 別忘了」才寫。
3. **刪除前先確認、衝突提示不覆蓋**。
4. **記憶要短、可行動、具體**：它是每次開場要讀的東西，不是日記，越精簡越可靠。
5. **身分覆寫**：某資料夾可讓 Claude 換名字/人設（財務夾當分析師、寫作夾當編輯）。
6. **記憶隔離**：子資料夾預設不繼承上層記憶，要帶哪些條目由使用者勾選。

---

## 8. 建資料夾

兩種方式搭配，同事**都不需手動一格一格建**：

| | 誰觸發 | 何時用 | 建什麼 |
|---|---|---|---|
| **一鍵批次**（指令，如 `begin-*`） | 手動跑一次 | 開新工作區 | 固定的標準資料夾骨架 |
| **隨需建立**（`subfolders` 技能） | 說「建一個 XX 資料夾」 | 臨時新專案/活動 | 單一新夾，帶標準 CLAUDE.md/MEMORY.md |

實務：**幾乎人人會用到**的放進批次指令；**因案而異**的留給隨需建立。

---

## 9. Onboarding 設計

若要在開機時做偏好問答，**區分兩層**，別混在一起：

| 層級 | 例子 | 該問每個人嗎 | 存哪 |
|---|---|---|---|
| **團隊/品牌層**（要一致） | 品牌 tone、禁用字、合規 | ❌ 由管理者設一次 | 根 `CLAUDE.md` |
| **個人層**（因人而異） | 名字、負責通路、回覆風格 | ✅ 可問 | 各自帳號偏好，或工作區 `CLAUDE.md` |

原則：**去劇場化、選項式、3～4 題內、答案直接寫進 `CLAUDE.md`（不要求貼設定頁）、可全跳過。**

---

## 10. 接真實數據

技能只「教 Claude 怎麼做」，**自己抓不到真實數據**。要接數據靠**連接器 / MCP**。

把「連接」拆成兩步：

| 步驟 | 在做什麼 | 能打包嗎 |
|---|---|---|
| **① 佈線/註冊** | 告訴 Claude「有這連接器、在哪、怎麼啟動」 | ✅ 可用 `.mcp.json` 自動 |
| **② 授權/登入** | 用**自己的帳號**登入服務 | ❌ 一定各自手動一次 |

- 憑證（API key / token）一律用**環境變數**帶入，**絕不寫死打包**（等於把帳密給全公司）。
- **CLI（gh / Vercel / Firecrawl…）**：plugin **不會安裝** system binary。要給 Claude 這些能力，**優先用它們的 MCP server**（Firecrawl/Vercel/GitHub 都有），而非包 CLI。
- **介面限制**：`.mcp.json` 是 Claude Code 機制；桌面/網頁無 shell，跑不了 `npx` 型 MCP，改用內建 Connectors。

---

## 11. 擴充

判斷準則：

> **同一群人、同一領域、要一起裝 → B（同 plugin 加技能）。**
> **不同群人/領域、要各自裝與更新 → C（同市集加第二個 plugin）。**

| | B：同 plugin 加技能 | C：市集加第二個 plugin |
|---|---|---|
| 動到的檔 | 只加 `skills/新技能/SKILL.md` | 加 `plugins/新plugin/` + 改 `marketplace.json` |
| 改 marketplace.json | ❌ 不用 | ✅ 多列一筆 |
| 安裝 | 跟著原 plugin 一起 | 同市集但**分開選裝** |
| 版本 | 併入原 plugin | 有自己的 version |

> plugin 的 skills 是**扁平**的，沒有資料夾分組——「新 section」＝多加一個 `SKILL.md`。

---

## 12. 維護與最佳實務

**加技能流程**（B）：
```bash
mkdir -p plugins/<plugin>/skills/<新技能>
#  寫 SKILL.md（最上方 description 就是觸發器）
claude plugin validate ./plugins/<plugin>
#  plugin.json version +1
git add . && git commit -m "feat(skill): <名>" && git push
#  同事按「更新」即同步
```

**慣例與注意**：
- **版本號**：修 bug → patch；加技能 → minor；破壞相容 → major。
- **觸發語別打架**：技能一多，`description` 觸發語要具體、不重疊，避免一句話命中多個技能。
- **產出格式固定**：各技能延續「固定輸出區塊 + 規範段」，全團隊產出才一致。
- **先驗證再 push**：養成 `claude plugin validate` 習慣。
- **技能不需登記進 manifest**：放進 `skills/` 自動掃描。

---

## 13. 安全性

- 純 `.md` 流程（技能/指令）風險低、行為透明；不含 hooks/執行檔就不會跑外部程式。
- 一旦接 MCP（HubSpot、廣告帳號…），Claude 即可讀寫真實系統 → 發佈前確認權限範圍與合規。
- 自訂 repo / Upload 的 plugin 不經審核 → 只裝信任來源的。
- 憑證永遠走環境變數，不進 git。

---

## 14. 決策速查表

| 情境 | 怎麼做 |
|---|---|
| 只是自己用、單一專案 | 用單獨 skill 放 `.claude/skills/`，不必做 plugin |
| 要分享給團隊、能自動更新 | 做成 **marketplace repo**，同事「Add from a repository」 |
| 臨時給一個人試 | **Upload plugin**（zip） |
| 加一個行銷技能 | **B**：加 `skills/xxx/SKILL.md`，不碰 marketplace.json |
| 另開一個部門工具 | **C**：加 `plugins/xxx/` + marketplace.json 一筆 |
| 要接廣告/CRM/爬蟲數據 | 打包 **MCP**（`.mcp.json`）佈線；憑證走 env；同事各自授權 |
| 桌面/網頁要接數據 | 走 **Connectors** 設定頁，各自登入（`.mcp.json` 不適用） |
| 跨 session 記住進度 | 用 **CLAUDE.md + MEMORY.md**（狀態外置、記憶隔離） |

---

## 15. 給同事的三句話安裝說明

貼進群組即可用。把 `<你的帳號>` 換成實際 GitHub 帳號。

> **① 安裝**：到 Claude → **Customize → Plugins → Add → Add marketplace → Add from a repository**，貼上 `https://github.com/<你的帳號>/marketing-os`，在清單找到 **marketing-os** 按 Install。
> **② 開工**：打開任一工作資料夾，輸入 `/marketing-os:begin-marketing` 鋪好標準結構；之後用自然語言使喚技能（「寫三則 IG」「做這週週報」），要記事就說「記一下…」。
> **③ 接數據（選）**：想讓競品分析即時爬網頁，先到 firecrawl.dev 申請 API key，設成環境變數 `FIRECRAWL_API_KEY`；沒設也能用，只是競品/數據改用你提供的資料。

**CLI 版（同事習慣終端機的話）**
```bash
/plugin marketplace add <你的帳號>/marketing-os
/plugin install marketing-os@marketing-os-marketplace
# 開工同上：/marketing-os:begin-marketing
```

> 提醒：若 repo 設為 private，同事的 GitHub 帳號需有讀取權限（或設一次 git 憑證）才裝得起來。

---

_本文件隨 repo 版本控管，可自由增修。_
