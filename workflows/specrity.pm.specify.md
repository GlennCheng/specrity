---
description: "PM 從 Jira Ticket 起草 PRD — 含 Clarify 互動、Draft-First 狀態機、跨 Session 持久化"
---

# /specrity.pm.specify <TICKET_ID>

## 設計理念

> **此流程專為非技術背景的 PM 設計。**
> 核心目的是讓 PM 專注於**任務的需求與目的**，避免干涉 RD 的邏輯架構與技術選用。
>
> **PM 的職責：** 釐清「使用者要什麼」、「為什麼要做」、「怎樣算成功」。
> **技術決策交給 RD：** 資料庫設計、框架選用、API 架構、前後端分工等，全部留給 `/specrity.dev.plan` 階段由工程師處理。
> **RD 有需要時主動回來問 PM**，而非由 PM 介入技術細節。

## 概述
此 workflow 協助 PM 從 Jira Ticket 建立產品需求文件（PRD）。基於原生 Specrity 的 specify + clarify 整合設計。採用 **Draft-First** 狀態機：PRD 草稿存在本地 `drafts/` 目錄進行 Clarify 互動，定稿後**自動發布到 Confluence**（PM 不需要碰 Git）。支援跨對話 session 無縫接續。

Clarify 採用**分輪互動 + Checkpoint**機制（源自原生 Specrity clarify），每輪 1~5 題關聯問題，每題附推薦答案。無硬性輪數上限，由 AI 評估覆蓋率 + PM 確認共同決定停止時機。PRD 產出後自動進行品質驗證（對照 `.specrity/templates/spec-template.md`）。

## 前置條件
- Atlassian MCP Server 已連線（若未連線，進入降級模式）

---

## 執行步驟

### Step 0: 載入專案設定

#### 0-A: 開場與上下文檢查 (Initialization & Context Check)
1. **印出歡迎橫幅**：一旦收到此指令，**第一句話**必須使用 Markdown 的 `text` 區塊，印出以下 ASCII Banner：
   ```text
      ____                  _ __       
     / __/___  ___ ________(_) /___ __ 
    _\ \/ _ \/ -_) __/ __/ / __/ // /  
   /___/ .__/\__/\__/_/ /_/\__/\_, /   
      /_/                     /___/    
   ```
2. **上下文檢測 (Context Sanity Check)**：
   - AI 掃描目前對話視窗 (Session) 的對話記錄。
   - 若發現記憶體中混雜了「其他 Ticket 或不相干任務」的上下文，必須暫停並警告：
     `⚠️ [Specrity 警告] 偵測到當前會話中包含其他任務的上下文，可能會導致生成的內容錯亂。建議另開全新的 Chat。但若確認是相關任務，可回覆繼續。`
   - 等待使用者確認。若上下文乾淨，則印出 `✅ Context check passed.` 並繼續。

#### 0-B: 版本檢查
1. 讀取專案根目錄的 `.specrity-installed`
2. 若檔案不存在 → 顯示：
   ```
   ⚠️ 未偵測到 specrity 安裝紀錄。
   請先執行安裝：~/specrity/install.sh <your-tool>
   ```
3. 若檔案存在，檢查 `installed_at` 時間戳記：
   - 若距今超過 30 天 → 顯示：
     ```
     💡 你的 specrity workflow 已安裝 N 天（v0.x.x）。
     建議更新：cd ~/specrity && git pull && ./install.sh <your-tool>
     ```
   - 若在 30 天內 → 靜默通過，不顯示任何訊息
4. 不管版本新舊，都繼續執行後續步驟（只是提醒，不阻擋）

#### 0-C: 載入設定

設定優先權：`.specrity/.specrity.yml` → `.env` → 自動偵測

1. 找到專案根目錄（往上層尋找 `.git`）
2. **強制讀取 `.specrity/.specrity.yml`**：
   - AI 必須確實解析此檔案，禁止憑空猜測。
   - 提取 `spec_mode`：`local`（預設）/ `submodule` / `external`
   - 提取 `spec_path`：spec 目錄的相對路徑（預設 `specrity/`）
   - 提取 `jira_transitions`：狀態轉移對應表（若存在）
   - 提取 `jira_cloud_id`（選填）
   - 提取 `jira_project_key`（選填）
   - **提取 `confluence_space_id`（必填）**
   - **提取 `confluence_parent_page_id`（選填）**
3. **Confluence 設定驗證**：
   - 若 `confluence_space_id` 為空或未設定 → **中止流程**並顯示：
     ```
     ❌ confluence_space_id 未設定！
     PRD 發布需要 Confluence Space ID。
     請在 .specrity/.specrity.yml 中設定 confluence_space_id 後重新執行。
     ```
4. 若 `spec_mode: external`，從 `.env` 讀取 `SPEC_REPO_PATH`
5. 若 `spec_mode: submodule`，**強制檢查 `.gitmodules`** 確認實際的 submodule 路徑是否與 `spec_path` 一致。
6. 若 `.specrity/.specrity.yml` 不存在且 `.env` 也沒設定：
   - 預設使用 `spec_mode: local`、`spec_path: specrity/`
7. 從 `<TICKET_ID>` 解析 project key（如 `HTGO2-123` → `HTGO2`）
8. 若未設定 `jira_cloud_id`，透過 MCP `getAccessibleAtlassianResources` 自動取得
9. 決定 spec 根目錄 `$SPEC_ROOT`：
   - `local` / `submodule` → `$PROJECT_ROOT/$spec_path`
   - `external` → `$SPEC_REPO_PATH`

#### 0-D: 載入個人設定檔 (User Preferences)

嘗試在專案根目錄讀取 `.specrity-user.yml`：
- 若存在，提取 `language` 設定，決定 AI 回覆與輸出文件的主要語言。
- 提取 `pm_card_features` 下的布林值（`show_recommendation`, `show_stakeholder_impact`, `show_effort_estimate`, `show_jira_quote`），這將控制後面「Clarify 卡片」的詳細程度。
- 若檔案不存在，套用預設值（全部為 `true`，語言預設 `zh-TW`）。

### Step 1: 狀態偵測（State Detection）

檢查 `$SPEC_ROOT/drafts/<TICKET_ID>/` 是否存在：

#### 情況 A：不存在 → NEW 模式
1. 從 Jira 擷取 Ticket 資訊：
   ```
   使用 MCP tool: getJiraIssue
   參數: cloudId = $JIRA_CLOUD_ID, issueIdOrKey = <TICKET_ID>
   ```
2. **深度上下文檢測 (Deep Context Sanity Check)**：
   - AI 必須對比「剛剛擷取到的 Jira Ticket 內容」與「先前的對話歷史記錄」。
   - 若發現歷史對話中的任務需求、功能描述或技術討論，**與本張 Ticket 完全無關甚至衝突**（AI 判斷將導致嚴重的幻覺或資訊混亂），必須中斷流程並拒絕推進：
     `❌ [Specrity 錯誤] 深度分析發現歷史對話（如前一項任務討論）與本次 Ticket 存在明顯衝突。強烈建議您開啟一個「全新乾淨的 Chat」後重新執行指令，以避免 PRD 內容失真。`
   - 除非使用者強行命令略過，否則停留在此階段。
3. 建立 `drafts/<TICKET_ID>/` 目錄
4. 將 Jira 內容儲存為 `jira-snapshot.md`
5. 初始化 `state.yml`：
   ```yaml
   ticket: <TICKET_ID>
   phase: drafting
   created_at: <timestamp>
   updated_at: <timestamp>
   clarify_rounds: 0
   prd_version: 0
   branch_name: null
   ```
6. 初始化空的 `clarify-log.md` 和 `prd.md`
7. **強制執行 MCP Jira 更新**（在開始發問之前）：
   > **[CRITICAL INSTRUCTION TO AI]**: 透過 MCP 執行 `addCommentToJiraIssue` 時，`commentBody` 參數 **MUST EXACTLY MATCH (必須完全一致，不可改寫)** 雙引號內的內容：
   ```
   使用 MCP tool: addCommentToJiraIssue
   參數: cloudId = $JIRA_CLOUD_ID, issueIdOrKey = <TICKET_ID>, commentBody = "🤖 [Specrity] ✍️ PRD 起草與釐清階段已開始"
   ```
8. **狀態轉移 (Jira Status Transition)**：
   > **[CRITICAL INSTRUCTION TO AI]**: 根據 `Step 0-C` 讀取到的 `jira_transitions` 中的 `specify_start` 設定（格式如 `"To Do -> Specifying"`）。若本 Ticket **當前狀態**與設定的**來源狀態**一致，請使用 MCP `getTransitionsForJiraIssue` 查詢對應 `id`，再呼叫 `transitionJiraIssue` 將其拖拉至目標狀態。若當前狀態不吻合則略過。
9. 進入 Step 2: Clarify

#### 情況 B：存在 + phase: drafting → RESUME 模式
1. 讀取 `state.yml` 取得目前進度
2. 讀取 `prd.md` 取得目前草稿
3. 讀取 `clarify-log.md` 取得過去的問答紀錄
4. 比對 Jira 最新狀態 vs `jira-snapshot.md`：
   - 若有差異，列出變更並提醒使用者
   - 更新 `jira-snapshot.md`
5. 向使用者報告：
   ```
   📋 「PROJ-123」PRD 進度恢復：
   - 狀態：起草中（第 N 輪 Clarify）
   - 上次更新：<timestamp>
   - 未回答的問題：N 個
   要繼續 Clarify 嗎？
   ```
6. 進入 Step 2: Clarify（從上次中斷處繼續）

#### 情況 C：存在 + phase: finalized → FINALIZE 模式
1. 讀取 `prd.md` 最終版
2. 詢問使用者：「PRD 已定稿但尚未發布到 Confluence，要 (1) 發布到 Confluence (2) 繼續修改？」
3. 若選擇發布 → 進入 Step 4
4. 若選擇修改 → 進入 Step 2

#### 情況 D：存在 + phase: published → PUBLISHED / REVISE 模式
1. 讀取本地 `prd.md` 和 `state.yml`
2. **🔍 Confluence 衝突偵測**：
   - 從 `state.yml` 取得 `confluence_page_id` 和 `confluence_last_version`
   - 使用 MCP `getConfluencePage`（`pageId = $confluence_page_id`）取得 Confluence 頁面 metadata
   - 若 MCP 回傳錯誤（頁面不存在）：
     ```
     ⚠️ Confluence 頁面已不存在（可能被刪除）。
     • 頁面 ID：{confluence_page_id}
     • 原始 URL：{confluence_url}

     選擇：
     1️⃣ 重新建立頁面 — 使用本地 prd.md 建立新頁面
     2️⃣ 中止 — 不做任何操作
     ```
     若選擇重新建立 → 清除 `state.yml` 中的 `confluence_page_id`，進入 Step 4
   - 比對 Confluence `version.number` 與 `state.yml` 的 `confluence_last_version`：
     - 若版本相同 → 無衝突，繼續
     - 若版本不同 → **展示衝突資訊**：
       ```
       ⚠️ Confluence 衝突偵測

       📄 Confluence 頁面已被他人修改：
       • 修改者：{lastModifiedBy.displayName}
       • 修改時間：{lastModifiedDate}
       • 版本變化：v{old} → v{new}（共 {diff} 次修改）

       📋 變更內容摘要：
       （AI 比對 Confluence 最新內容 vs 本地 prd.md，用 diff 高亮標示差異段落）

       👉 請選擇：
       1️⃣ 以 Confluence 版本為準 — 下載線上版覆蓋本地草稿，然後繼續修改
       2️⃣ 以本地版本為準 — 忽略線上修改，發布時覆蓋 Confluence
       3️⃣ 手動合併 — AI 協助將兩版內容合併，產出合併版供確認
       4️⃣ 中止 — 不做任何操作
       ```
       - 選項 1️⃣：使用 MCP `getConfluencePage`（`contentFormat = "markdown"`）下載內容覆蓋本地 `prd.md`，更新 `state.yml` 版本記錄
       - 選項 2️⃣：跳過下載，繼續正常流程（發布時會記錄「覆蓋了線上修改」的 Jira comment）
       - 選項 3️⃣：AI 產出合併版，展示 diff 給 PM 確認後覆蓋 `prd.md`
       - 選項 4️⃣：結束流程
3. 提供 Confluence 頁面連結並詢問：「PRD 已發布到 Confluence，要進入修改模式嗎？」
   - Confluence URL：`{confluence_url}`
4. 若要修改，讀取最新 Jira comment（可能有 Dev 回饋）
5. 進入 Step 2: Clarify（修改模式）

### Step 2: Clarify 互動

#### 2-A: 初始分析

1. **必須實際讀取 Template 檔案**：使用檔案讀取工具，去讀取 `$PROJECT_ROOT/.specrity/.specrity/templates/spec-template.md` 檔案的完整內容，理解 PRD 應有的完整結構與限制
2. 基於 Jira ticket 內容 + 現有 prd.md 草稿，進行**結構化覆蓋掃描**：
   - 對照 spec-template 的每個區塊，標記覆蓋狀態：`Clear` / `Partial` / `Missing`
   - 將 `Partial` 或 `Missing` 的區塊列為候選釐清問題

3. **解法建議偵測**：掃描 Jira ticket 的 description 和 comments，識別「具體解法建議」vs「確認過的要求」：
   - 若偵測到客戶或 PM 提出的具體實作建議或解法（如：「加一個 CSV 匯出按鈕」、「幫我用 Elasticsearch」、「這裏要一個 popup」），
     **不得直接寫成 FR**，而是：
     - 記錄到 PRD 的「原始解法建議」區塊
     - **自動產生一題 Clarify 問題**，詢問 PM 這是：
       - A: 這是必須執行的確認需求（Must-have） → 轉化為 FR，並釐清其背後目的
       - B: 僅是初步的解法建議 → 留在「原始解法建議」供參考，PRD 應聚焦於要解決的核心問題
       - C: 需要 RD 做 RCA 後再決定最佳解法 → 標記 `[NEEDS RCA]`，留給 `/specrity.dev.plan` 處理

4. ⚠️ **重要限制：你是 PM 的產品思維夥伴，不是工程師。絕對不要詢問技術實作細節。**

   **提問維度（依序檢視，挑出最需要釐清的）：**

   | 維度 | 說明 | 範例問題 |
   |------|------|---------|
   | 🎯 故事背景與目的 | 為什麼要做？解決什麼痛點？ | 「這個改動要解決的核心問題是什麼？現況的痛點是？」 |
   | 👤 使用者是誰 | 主要對象、角色差異 | 「這個功能主要服務哪類使用者？訪客和會員有差異嗎？」 |
   | 🔄 使用者行為 | User flow、操作路徑 | 「使用者進入這個頁面後，最理想的操作路徑是什麼？」 |
   | 📱 裝置與場景 | 桌面/手機、使用情境 | 「手機版的選單行為和桌面版一樣嗎？還是有不同的互動方式？」 |
   | 🔀 邊界條件 | 例外狀況、極端情境 | 「如果選單項目超過 50 個，畫面應該怎麼呈現？」 |
   | ⚡ 體驗期望值 | 速度、流暢度、即時性 | 「使用者展開子選單時，可以接受短暫的載入提示嗎？還是必須秒開？」 |
   | 📊 成功指標 | 怎樣算做好了 | 「這個需求上線後，你會用什麼指標衡量成功？」 |
   | ⚠️ 失敗處理 | 出錯時使用者看到什麼 | 「如果資料載入失敗，使用者應該看到什麼？空白？還是預設內容？」 |
   | 🔒 權限與可見性 | 誰能看、誰不能看 | 「這個區塊對未登入使用者也可見嗎？不同角色看到的內容有差嗎？」 |
   | 📦 範圍與分期 | MVP vs 完整版 | 「第一階段必須包含哪些功能？哪些可以之後再做？」 |

   **將技術問題轉譯為需求問題（重要技巧）：**

   | ❌ 不要這樣問（技術） | ✅ 應該這樣問（需求） |
   |----------------------|---------------------|
   | 要用 GraphQL 還是 REST API？ | 選單內容需要即時更新？還是每次部署更新就好？ |
   | 要用 lazy loading 嗎？ | 使用者展開子選單時，可以接受短暫的載入動畫嗎？ |
   | 要用 SSR 還是 CSR？ | 這個頁面的首次載入速度很重要嗎？SEO 是考量嗎？ |
   | 資料庫 schema 要怎麼設計？ | 這些設定需要跨裝置同步嗎？ |
   | 要用 WebSocket 嗎？ | 需要「即時」看到其他人的變更嗎？還是重新整理就好？ |
   | 要做 component 拆分嗎？ | 這個區塊未來會在其他頁面重複使用嗎？ |
   | 要支援 i18n 嗎？ | 這個功能需要支援多語系嗎？目前有哪些語言？ |

4. 產出候選問題，根據 **Impact × Uncertainty** 排序，按維度分組

### Step 2: Clarify 互動

（...前略 2-A 分析部分...）

#### 2-B: 分輪互動 + Checkpoint（每輪 1~5 題關聯問題）

**每一輪（Round）將同維度或有因果關係的問題分組，每輪 1~5 題。無硬性輪數上限，由 AI 評估 + PM 確認共同決定何時停止。**

每輪流程：

1. **分組與排版要求**：
   > **[CRITICAL INSTRUCTION TO AI]**: 你接下來的輸出內容 **MUST STRICTLY FOLLOW (必須嚴格遵守)** 以下「Modern Dashboard (現代卡片)」的 Markdown 排版格式與結構，不要隨意改變。必須使用引言區塊（`> `）、Emoji 與分隔線，讓畫面乾淨俐落。標題必須加上目前的 Round 數。

   2. **究極版多選題格式範例**：
      > **[CRITICAL INSTRUCTION TO AI]**: 根據 `0-D` 讀取到的 `pm_card_features` 開關，決定是否輸出對應的區塊。如果全部為 `true`，請嚴格套用以下 Markdown 排版：
      ```markdown
      > 🎯 **Specrity Clarifier** | <TICKET_ID>
      > 
      > ---
      > 
      > 💡 **<維度名稱>釐清 (Round <N> - 問題 <X>/<Y>)：<問題簡述>**
      > 
      > <details>
      > <summary>📜 <b>檢視 Jira 相關原文</b></summary>
      > 
      > > *<如果 show_jira_quote=true，請在此引用觸發該問題的 Jira 原文>*
      > </details>
      > 
      > **可選方案：**
      > 
      > - 1️⃣ **<選項一簡稱>**
      >   - 📝 **說明：** <詳細描述具體行為>
      >   - ⚙️ **實作成本：** <如果 show_effort_estimate=true，標示 T-shirt 尺寸 [Size S/M/L] 及粗估天數，並附技術理由>
      >
      > - 2️⃣ **<選項二簡稱>**
      >   - 📝 **說明：** <詳細描述具體行為>
      >   - ⚙️ **實作成本：** <同上>
      >
      > - 3️⃣ **<選項三簡稱>**
      >   - 📝 **說明：** <詳細描述具體行為>
      >   - ⚙️ **實作成本：** <同上>
      > 
      > ---
      > 
      > 🏆 **AI 綜合分析與推薦：** <如果 show_recommendation=true，請給出獨立的推薦選項與理由>
      > 
      > ⚖️ **利害關係人影響 (Stakeholders Impact)：** <如果 show_stakeholder_impact=true，請列出>
      > - 🧑‍💻 **RD 開發端**：<對開發成本、維護性的影響>
      > - 👤 **終端使用者**：<對 UX 的好壞>
      > - 🧑‍💼 **業務/客戶端**：<對商業目標或 KPI 的影響>
      > 
      > ---
      > 
      > 💬 *請回覆選項數字，或是輸入您的其他設計考量。*
      ```

   3. **簡答題格式範例**（無離散選項時）：
      ```markdown
      > 🎯 **Specrity Clarifier** | <TICKET_ID>
      > 
      > ---
      > 
      > 💡 **<維度名稱>釐清 (Round <N> - 問題 <X>/<Y>)：<問題簡述>**
      > 
      > <details>
      > <summary>📜 <b>檢視 Jira 相關原文</b></summary>
      > 
      > > *<同上，若開啟引述功能>*
      > </details>
      > 
      > **強烈建議做法 🏆：**
      > - 📝 **說明：** <AI 根據經驗或工程最佳實踐給出的具體建議>
      > - ⚙️ **實作成本：** <同上，若開啟工時評估功能>
      > 
      > ⚖️ **利害關係人影響 (Stakeholders Impact)：**
      > <同上，若開啟利害關係人功能>
      > 
      > ---
      > 
      > 💬 *您可以接受建議（回覆 `Yes`），或提供你自己的答案。*
      ```

3. **Checkpoint（每輪結束後 — ⚠️ 強制執行）**：

> [!CAUTION]
> **每一輪結束後，MUST 執行 Checkpoint。絕對不可跳過。**
> **在 PM 明確說出「定稿」或「夠了」之前，不得進入 Step 3。**

Checkpoint 展示格式：

> **[CRITICAL INSTRUCTION TO AI]**: 此 Checkpoint 畫面 **MUST STRICTLY FOLLOW** 以下 Modern Dashboard 風格排版，絕不准使用舊版純文字格式：

```markdown
   > 📊 **Clarify Checkpoint** | Round <N> 完成
   > 
   > ---
   > 
   > - ✅ **已釐清：** <列出已經確認的維度，如：故事背景、範圍>
   > - ❓ **尚未確認：** <列出還缺少的維度，如：使用者角色、成功指標>
   > 
   > 📈 **當前覆蓋率：** X / Y 主要面向已釐清
   > 
   > 👉 還有 <Z> 個面向不太清楚，要繼續嗎？
   > - 回覆 `繼續` 或 直接按 `Enter` → 進入下一輪 (Round <N+1>)
   > - 回覆 `定稿` 或 `夠了` → 結束詢問，進入 PRD 產出階段
```

**Checkpoint 決策邏輯：**
- PM 說「繼續」→ **回到 2-B 開始下一輪**（MUST 再次出題 + 再次 Checkpoint）
- PM 說「定稿」/「夠了」/「可以了」/「done」→ 進入 Step 3
- **若 PM 沒有明確回應停止，MUST 繼續下一輪**

#### 2-C: 答案處理 + 持久化

使用者回答後：
1. 若回覆「yes」、「推薦」、「建議」→ 採用推薦/建議的答案
2. 若模糊 → 追問一次澄清（不計入 Round）
3. **立即**記錄到 `clarify-log.md`：
   ```markdown
   ## Round N — <維度名稱> — <date>

   **Q1: <問題>**
   **推薦：** <推薦答案>
   **A：** <使用者最終答案>

   **Q2: <問題>**
   **推薦：** <推薦答案>
   **A：** <使用者最終答案>

   **覆蓋率：** N/M 主要面向已釐清
   **PM 決定：** 繼續 / 定稿
   ```
4. **立即**更新 `prd.md` 草稿中對應的區塊
5. 更新 `state.yml` 的 `clarify_rounds` 和 `updated_at`
6. **執行 Checkpoint**（回到上方第 3 步）

### Step 3: 產出 PRD + 品質驗證

#### 3-A: 產出完整 PRD

1. **必須實際讀取 Template 檔案**：使用檔案讀取工具，去讀取 `$PROJECT_ROOT/.specrity/.specrity/templates/spec-template.md` 檔案的完整內容，作為產出 PRD 的結構參照
2. 根據所有 Clarify 資訊 + Jira snapshot，填充 template 的每個區塊
3. 對於仍不清楚但影響不大的地方，使用合理預設值並記錄在 **Assumptions** 區塊
4. 對於仍不清楚且影響重大的地方，標記 `[NEEDS CLARIFICATION: 具體問題]`（最多 3 個）
5. 儲存到 `drafts/<TICKET_ID>/prd.md`
6. 更新 `state.yml` 的 `prd_version` + 1

#### 3-B: 品質驗證 + 矛盾偵測（自動檢查）

**品質檢查**（對照 spec-template）：

- **完整性**：所有 mandatory 區塊都填寫完成？
- **清晰度**：沒有模糊形容詞（「快速」、「直覺」）未量化？
- **可測性**：每個 FR 和 SC 都可以被客觀驗證？
- **PM 語言**：沒有技術術語（框架、API、資料庫）混入？
- **`[NEEDS CLARIFICATION]` 數量**：不超過 3 個？

品質問題由 AI 自動修正（最多 2 輪）。

**矛盾偵測**（關鍵步驟）：

交叉比對 PRD 各區塊，偵測以下幾類矛盾：

| 矛盾類型 | 範例 |
|---------|------|
| 範圍矛盾 | Scope 限定「僅 Desktop」，但 FR-003 描述了 Mobile 行為 |
| 需求衝突 | FR-001 要求「即時更新」，但 Experience 寫「可接受延遲」 |
| 優先級矛盾 | User Story P1 依賴 P3 的功能才能運作 |
| 指標矛盾 | SC-001 期望「2 秒內載入」，但 FR 要求「載入所有子選單資料」 |
| 假設衝突 | Assumptions 假設「既有登入不變」，但 FR 新增了角色權限 |

**偵測到矛盾時的處理流程：**

```
產出 PRD → 品質檢查 → 矛盾偵測
                         │
                    有矛盾？
                    │      │
                    否      是
                    ↓      ↓
               3-C 展示   列出矛盾，回到 Clarify：
                          ⚠️ PRD 中偵測到 N 個矛盾：
                          
                          矛盾 1: 範圍限定「僅 Desktop」，
                                  但 FR-003 描述了 Mobile 行為。
                          → 推薦：移除 FR-003 或調整範圍
                          
                          矛盾 2: ...
                          
                          → PM 逐一決定如何修正
                          → AI 更新 PRD
                          → 重新執行矛盾偵測
                          → 直到無矛盾 → 3-C
```

矛盾提問格式與 Clarify 相同（推薦答案 + 選項表格），PM 可選擇或自由回答。

#### 3-C: 展示與確認

1. 展示完整 PRD 給使用者確認（此時已通過品質檢查 + 無矛盾）
2. 若有 `[NEEDS CLARIFICATION]` 標記，逐一展示並讓使用者選擇：
   - 提供答案 → 更新 PRD
   - 留給 RD 處理 → 保留標記
3. 使用者確認後，更新 `state.yml` phase 為 `finalized`

### Step 4: 定稿與發布到 Confluence

PM 確認 PRD 後，統一發布到 Confluence。**不再需要 Git 操作或建立 Branch。**

#### 4-A: 發布前衝突偵測

> **[CRITICAL INSTRUCTION TO AI]**: 此步驟僅在 `state.yml` 已有 `confluence_page_id` 時執行（即更新已發布的 PRD）。若為首次發布，跳到 4-B。

1. **Confluence Review Comment 檢查**：
   - 使用 MCP `getConfluencePageInlineComments`（`pageId = $confluence_page_id`, `resolutionStatus = "open"`）
   - 若有未解決的 comment：
     ```
     ⚠️ PRD 的 Confluence 頁面上有 N 個未解決的 Review Comment：
     1. @{author}: "{摘要}"
     2. @{author}: "{摘要}"

     建議先處理這些 Comment 再重新發布。
     (1) 我已經處理了，繼續發布
     (2) 中止
     ```
   - 若無 → 通過，繼續後續步驟
2. 使用 MCP `getConfluencePage`（`pageId = $confluence_page_id`）取得最新 metadata
3. 若 MCP 回傳錯誤（頁面不存在）：
   ```
   ⚠️ Confluence 頁面已不存在（可能被刪除）。
   • 頁面 ID：{confluence_page_id}
   選擇：
   1️⃣ 重新建立頁面 — 使用本地 prd.md 建立新頁面
   2️⃣ 中止
   ```
   若選擇重新建立 → 清除 `state.yml` 中的 `confluence_page_id`，繼續 4-B
4. 比對 Confluence `version.number` 與 `state.yml` 的 `confluence_last_version`
5. 若版本不同 → 展示衝突資訊（格式同 Step 1 情況 D 的衝突偵測）
   - 選項 1️⃣ 以 Confluence 為準：下載後合併到 prd.md，**重新進入 Step 3-C 展示給 PM 確認**
   - 選項 2️⃣ 以本地為準：繼續 4-B（發布時會覆蓋 Confluence）
   - 選項 3️⃣ 手動合併：AI 產出合併版，確認後繼續 4-B
   - 選項 4️⃣ 中止
6. 若版本相同 → 無衝突，繼續 4-B

#### 4-B: 發布到 Confluence

1. 讀取 `drafts/<TICKET_ID>/prd.md` 的完整內容
2. 檢查是否已有 Confluence page（從 `state.yml` 的 `confluence_page_id` 判斷）：
   - **若無（首次發布）** → 使用 MCP `createConfluencePage`：
     ```
     使用 MCP tool: createConfluencePage
     參數:
       cloudId = $JIRA_CLOUD_ID
       spaceId = $CONFLUENCE_SPACE_ID
       parentId = $CONFLUENCE_PARENT_PAGE_ID（若有設定）
       title = "[<TICKET_ID>] <Ticket Title> — PRD"
       body = prd.md 內容
       contentFormat = "markdown"
     ```
   - **若有（更新發布）** → 使用 MCP `updateConfluencePage`：
     ```
     使用 MCP tool: updateConfluencePage
     參數:
       cloudId = $JIRA_CLOUD_ID
       pageId = $confluence_page_id
       title = "[<TICKET_ID>] <Ticket Title> — PRD"
       body = prd.md 內容
       contentFormat = "markdown"
       versionMessage = "Updated via Specrity"
     ```
3. **Confluence MCP 連線失敗的降級處理**：
   - 若 MCP 無法連接 Confluence：
     ```
     ⚠️ 無法連接 Confluence。PRD 已定稿並存在本地。
     選擇：
     1️⃣ 稍後重試 — 結束流程，下次執行指令時會再嘗試發布
     2️⃣ 匯出 PRD — 將 prd.md 內容展示在畫面上，您可手動貼到 Confluence
     ```
   - 若選擇稍後重試 → 保持 `phase: finalized`，結束流程
   - 若選擇匯出 → 展示 PRD 全文，結束流程
4. 將 Confluence 回傳的 page ID、URL、版本號記錄到 `state.yml`：
   ```yaml
   confluence_page_id: <page_id>
   confluence_url: <page_url>
   confluence_last_published_at: <timestamp>
   confluence_last_version: <version.number>
   ```
5. 更新 `state.yml`：`phase: published`

#### 4-C: 更新 Jira Ticket（綁定 Confluence 連結）

> **[CRITICAL INSTRUCTION TO AI]**: 此步驟至關重要。Confluence 連結必須寫入 Jira Ticket，讓 RD 端能直接從 Ticket 找到 PRD，無需依賴本地檔案。

1. **將 Confluence URL 寫入 Jira Ticket 欄位**：
   > 嘗試使用 MCP `editJiraIssue` 將 Confluence URL 寫入 Jira ticket 的描述或自訂欄位：
   ```
   使用 MCP tool: editJiraIssue
   參數:
     cloudId = $JIRA_CLOUD_ID
     issueIdOrKey = <TICKET_ID>
     fields = {
       // 嘗試以下方式之一（依 Jira 專案設定而定）：
       // 方式 A：若有自訂欄位（如 "PRD Link" 或 "Confluence Link"），使用該欄位
       // 方式 B：若無自訂欄位，在 description 最前面加註 Confluence 連結
     }
   ```
   - 若更新失敗（無權限或欄位不存在），不阻擋流程，僅記錄警告然後繼續

2. **在 Jira 加上 comment**：
   > **[CRITICAL INSTRUCTION TO AI]**: 透過 MCP 執行 `addCommentToJiraIssue` 時，`commentBody` 參數 **MUST STRICTLY FOLLOW (必須嚴格遵守)** 以下排版，將 `{confluence_url}` 替換為實際 URL：
   ```markdown
   > 🤖 **[Specrity] ✅ PRD 已發布到 Confluence**
   >
   > ---
   >
   > 📄 **Confluence 連結：** {confluence_url}
   > 🕐 **發布時間：** {timestamp}
   >
   > 工程師可執行 `/specrity.dev.plan <TICKET_ID>` 開始規劃。
   ```

3. **智慧 Reviewer 通知（Tech Lead @mention）**：
   > 根據 `.specrity.yml` 的 `reviewer_account_id` 和 `review_label` 設定。

   - **情況 A：Ticket 有 `review_label`（如 `needs-tech-review`）**
     - 若有設定 `reviewer_account_id` → **自動**在 Jira 加一則 comment @mention reviewer：
       ```
       > 🔔 @{reviewer} 此 PRD 已標記需要技術主管 Review。
       > 📄 Confluence: {confluence_url}
       ```
   - **情況 B：Ticket 沒有 `review_label`**
     - AI **主動評估**此 PRD 是否建議 Tech Lead Review：
       - 評估標準：涉及架構變更、跨系統整合、安全性、效能關鍵路徑、資料庫 schema 變更、新技術引入等
       - 若 AI 判斷建議 Review → 提示 PM：
         ```
         💡 AI 評估此 PRD 涉及較複雜的技術範圍（{原因摘要}），
         建議通知技術主管確認。
         (1) 是，幫我在 Jira @mention 主管
         (2) 不需要，跳過
         ```
       - 若 PM 選擇通知 → 加上 @mention comment（同情況 A）
       - 若 AI 判斷不需要 → 跳過，不提示
   - 若未設定 `reviewer_account_id` → 跳過整個步驟

4. **狀態轉移 (Jira Status Transition)**：
   > **[CRITICAL INSTRUCTION TO AI]**: 根據 `Step 0-C` 讀取到的 `jira_transitions` 中的 `specify_done` 設定（格式如 `"Specifying -> Spec Review"`）。若本 Ticket **當前狀態**與設定的**來源狀態**一致，請使用 MCP `getTransitionsForJiraIssue` 查詢對應 `id`，再呼叫 `transitionJiraIssue` 將其拖拉至目標狀態。若當前狀態不吻合則略過。
5. **完成提示**：
   ```
   ✅ PRD 已發布到 Confluence。
   📄 Confluence: {confluence_url}
   🔗 Jira Ticket 已更新（Confluence 連結已綁定）
   工程師可執行 /specrity.dev.plan <TICKET_ID> 開始規劃。
   ```

---

## Jira 降級處理

若 Atlassian MCP Server 連線失敗或 tool 呼叫失敗：

```
⚠️ Unable to connect to Jira MCP Server.
Please provide the following manually:

1. Ticket title:
2. Ticket description:
3. Acceptance Criteria:
4. Additional info:

(Paste and press Enter to continue)
```

後續步驟照常進行，只是跳過 Jira 狀態更新和 comment 操作。

Confluence 發布不受 Jira 降級影響（兩者獨立連線）。

---

## 預期產出

- `drafts/<TICKET_ID>/state.yml` — 狀態追蹤（含 Confluence page ID、版本號）
- `drafts/<TICKET_ID>/jira-snapshot.md` — Jira 需求快照
- `drafts/<TICKET_ID>/clarify-log.md` — Clarify 問答記錄
- `drafts/<TICKET_ID>/prd.md` — PRD 完整文件
- **Confluence 頁面** — PRD 正式發布版（`[TICKET_ID] Title — PRD`）
- Jira 更新（comment + 狀態 transition + Confluence 連結）
