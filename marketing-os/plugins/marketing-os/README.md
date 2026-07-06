# marketing-os

行銷部門的 Claude 作業系統。同仁安裝後，即可一鍵鋪好標準工作資料夾，並用自然語言觸發各項行銷技能。

## 包含內容

**`/begin-marketing` 指令** — 一鍵初始化
先做一段**輕量 onboarding**（3～4 題選項式）：設定品牌 tone／合規（團隊層）與你負責的通路／回覆風格（個人層），答案直接寫進 `CLAUDE.md`——**不要求你去貼任何設定頁**。接著建立根工作區與四個標準資料夾（活動企劃 / 社群經營 / 廣告投放 / 競品與市場），每個都附 `CLAUDE.md` 與 `MEMORY.md`。一個工作區只需跑一次。

> 品牌/合規屬**全團隊一致**的規範，建議由主管設定一次；一般同仁不確定可先跳過，沿用團隊既有規範。

**技能（講一句話自動觸發，全資料夾可用）**
| 技能 | 觸發範例 |
|---|---|
| `campaign-brief` | 「幫我寫活動企劃」 |
| `social-post` | 「來三則 IG 貼文」 |
| `ad-copy` | 「寫 Meta 廣告文案，A/B」 |
| `competitor-scan` | 「分析一下這幾家競品」 |
| `weekly-report` | 「做這週的行銷週報」 |
| `edm` | 「寫一封 EDM / 電子報」 |
| `subfolders` | 「建一個雙11活動的資料夾」（臨時新增單一資料夾，帶標準結構） |

## 建資料夾：兩種方式
- **一鍵鋪標準骨架** → `/marketing-os:begin-marketing`（開新工作區時跑一次，建好固定的四個標準夾）。
- **臨時新增單一資料夾** → 直接說「建一個 XX 資料夾」，`subfolders` 技能會做簡短訪談後建好，並保證帶標準 `CLAUDE.md`+`MEMORY.md`。

## 使用方式
1. 打開一個工作資料夾。
2. 輸入 `/marketing-os:begin-marketing` 鋪好結構。
3. 進任一資料夾，用自然語言使喚技能；要記事就說「記一下…」；要新夾就說「建一個…資料夾」。

## 重要觀念
- **技能是全域的**：裝好後所有資料夾都能觸發，不綁特定資料夾。
- **資料夾負責隔離**：各資料夾的 `MEMORY.md`（記憶）、`CLAUDE.md`（指令/身分）彼此獨立，不互相污染。

## 接上真實數據（連接器 / MCP）— 建議設定
本外掛的技能只負責「流程與產出」，本身**不會自動抓廣告/分析數據**。要讓 `weekly-report`、`competitor-scan` 用到真實數字，請在各自的 Claude 環境**連接對應的連接器**，例如：
- 廣告/分析成效（如 Supermetrics 類數據源）
- HubSpot（名單/CRM）
- Google Drive（素材/文件）
- Canva（出圖）

設定方式：
- **claude.ai / 桌面 App**：到「設定 → Connectors（連接器）」各自授權登入。
- **Claude Code**：可在 plugin 內放一份 `.mcp.json` 打包 MCP 伺服器，或用 `claude mcp add` 設定。

> 注意：連接器通常需要**每位同仁各自用自己的帳號授權一次**，這步無法由外掛代勞。技能已寫明「沒有真實數據來源時不編造數字」，所以未連接時它會請你提供數字或改用既有記錄。

### 已打包：Firecrawl（強化 competitor-scan）
本外掛內建一份 `.mcp.json`，已**佈線**好 Firecrawl MCP，讓 `competitor-scan` 能實際爬取競品官網／落地頁取得即時內容。

- **佈線（自動）**：在 **Claude Code** 安裝本外掛後，Firecrawl MCP 即註冊完成，同仁不需手動設定伺服器。
- **授權（各自手動，必做一次）**：每位同仁需設定自己的 `FIRECRAWL_API_KEY` 環境變數（到 firecrawl.dev 申請）。未設定時 `competitor-scan` 會自動退回「依既有知識推估」並提示去設定。
- **介面限制**：`.mcp.json` 屬 Claude Code 機制。**桌面/網頁（Cowork）沒有通用 shell、跑不了 `npx` 型 MCP**，那邊請改用內建連接器或忽略此段。

> 想再加 **Vercel / GitHub** 等？它們也有各自的 MCP server，作法相同：在 `.mcp.json` 的 `mcpServers` 多加一筆、憑證用環境變數帶入（**切勿把 token 寫死打包**）。gh / Vercel 偏開發工具，行銷情境通常用不到。

## 安全性
- 技能全為本地 `.md` 流程，不含 hooks / 可執行檔。
- 一旦連接 HubSpot、廣告帳號等，Claude 即可讀寫真實系統，請確認權限範圍與部門合規後再全員推行。
