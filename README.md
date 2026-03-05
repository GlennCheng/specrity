# Specrity

> Spec-Kit Jira Integration — 透過 AI Agent 自動化 Jira 工作流的 Workflow 套件。

## 概述

Specrity 為 PM 和工程師提供 4 個核心 AI Agent 指令，串接 Jira 與 Spec-Kit 的 PRD 管理流程：

| 指令 | 角色 | 說明 |
|------|------|------|
| `/specrity.pm.specify` | PM | 從 Jira Ticket 起草 PRD → Clarify → 建 Branch → 更新 Jira |
| `/specrity.dev.plan` | Engineer | 讀 PRD → 分析程式碼 → 產出 `plan.md` |
| `/specrity.dev.tasks` | Engineer | 讀 plan + spec → 拆解 tasks → 建 Jira Sub-tasks |
| `/specrity.dev.implement` | Engineer | 逐步實作 → 自動勾進度 → 更新 Jira |

## 特色

- **Draft-First 設計**：PM 的 PRD 草稿先存在 `drafts/` 目錄，定稿後才建 feature branch
- **跨 Session 持久化**：所有狀態（Clarify 問答、PRD 草稿）持久化到檔案系統，不同對話 session 可無縫接續
- **Jira 降級處理**：MCP 連線失敗時自動提示手動貼上需求
- **17+ 工具支援**：一鍵部署到 Claude Code、Cursor、Windsurf、Antigravity 等主流 AI 工具

---

## 部署方式

Specrity 採用 **「個人安裝」** 模式：workflow 檔案只存在使用者本機，不 commit 到專案 repo。
想用的人自行安裝，不影響不使用的團隊成員。

> 💡 之後如果團隊決定全面採用，可以改為 commit workflow 到 repo，只需跑一次 install.sh + git add。

---

## 初始設定（Lead 做一次）

Lead 只需要設定**專案層級的共用設定**（`.specrity.yml` + spec submodule），不需要部署 workflow。

### Step 1: 建立 Spec Submodule（推薦）

```bash
# 在 GitHub/GitLab 上建一個新 repo，例如 my-project-specs
# 然後在主專案中加為 submodule：

cd ~/my-project
git submodule add git@github.com:your-org/my-project-specs.git specs/
git commit -m "chore: add spec submodule"
```

### Step 2: 建立 `.specrity.yml`

在專案根目錄建立設定檔：

```yaml
# .specrity.yml — commit 到 repo
spec_mode: submodule
spec_path: specs/
```

或由 `install.sh` 互動式生成（安裝 workflow 時會自動詢問）。

### Step 3: Commit 共用設定

```bash
git add .specrity.yml
git commit -m "chore: add specrity project config"
git push
```

> **只有 `.specrity.yml` 和 submodule 會 commit 到 repo。Workflow 檔案不 commit。**

---

## 個人安裝（每個想用的人做一次）

### Step 1: Clone specrity

```bash
# 建議放在固定位置，之後更新也方便
git clone https://github.com/your-org/specrity.git ~/specrity
```

### Step 2: 在你的專案中安裝 workflow

```bash
cd ~/my-project

# 安裝到你使用的 AI 工具
~/specrity/install.sh agy            # Antigravity
~/specrity/install.sh cursor-agent   # Cursor IDE
~/specrity/install.sh claude         # Claude Code CLI
~/specrity/install.sh --help         # 查看所有支援的工具
```

`install.sh` 會：
1. 偵測專案根目錄
2. 讀取 `.specrity.yml`（如果 Lead 已設定）或互動式建立
3. 部署 workflow 到你的 AI 工具對應目錄（只在本機）

### Step 3: 確認 Atlassian MCP Server

在你的 AI 工具中確保 `atlassian-mcp-server` 已連線。

### Step 4: 開始使用！

```
/specrity.pm.specify HTGO2-123     ← 開始起草 PRD
/specrity.dev.plan HTGO2-123       ← 產出實作計畫
/specrity.dev.tasks HTGO2-123      ← 拆解任務 + 建 Jira Sub-tasks
/specrity.dev.implement HTGO2-123  ← 逐步實作
```

### ⚙️ Workflow 更新

specrity 的 workflow 版本更新時：

```bash
cd ~/specrity && git pull         # 更新 specrity
cd ~/my-project
~/specrity/install.sh agy         # 重新部署到你的工具
```

### 🗑️ 不想用了？

直接刪除對應檔案即可，不影響任何人：

```bash
# Antigravity
rm -rf ~/.agents/workflows/specrity.pm.specify.md  # etc.

# Cursor
rm .cursor/rules/specrity.mdc

# Claude Code
# 手動移除 CLAUDE.md 中的 specrity 區塊
```

---

## Spec 存放模式

### 🔥 推薦：Git Submodule（團隊使用）

PRD 存放在獨立 repo，以 submodule 掛進主專案。

```yaml
# .specrity.yml
spec_mode: submodule
spec_path: specs/
```

**好處：**
- PM 和 Engineer 權限分離
- PRD 版本控制和代碼獨立
- Review 分離

**團隊成員加入時：**
```bash
git clone --recurse-submodules git@github.com:your-org/my-project.git

# 或已 clone 過：
git submodule update --init --recursive
```

**安裝後的專案結構：**
```
my-project/
├── .specrity.yml              # ← commit（共用設定）
├── specs/                         # ← commit（submodule）
│   └── drafts/
│       └── HTGO2-123/
│           ├── state.yml
│           ├── clarify-log.md
│           └── prd.md
├── .agents/workflows/             # ← 不 commit（個人安裝的 workflow）
├── src/
└── .gitignore
```

**日常操作：**
```bash
# PM 起草 PRD（在 AI 工具中）
/specrity.pm.specify HTGO2-123

# 同步 spec submodule
cd specs && git pull origin main && cd ..
git add specs && git commit -m "chore: update specs"
```

**注意事項：**

| 情境 | 處理方式 |
|------|---------|
| 忘記 `submodule update` | Workflow 會提示執行 `git submodule update --init` |
| PM 在 spec repo push 後 | 主專案需 `git add specs && git commit` 更新 ref |
| 多人編輯同一個 PRD | 建議一個 ticket 只有一個 PM 負責 |

<details>
<summary>📁 其他模式</summary>

**Local 模式（最簡單，個人使用）**

```yaml
spec_mode: local
spec_path: specs/
```

PRD 直接存在專案子目錄，不需額外 repo。

**External 模式（特殊情境）**

```yaml
spec_mode: external
```

需在 `.env` 設定 `SPEC_REPO_PATH`（各人不同，不 commit）。
</details>

---

## 運作原理

### 為什麼用 Submodule？

```
主專案 repo（工程師管理）          Spec repo（PM + 工程師共用）
┌─────────────────────────┐       ┌──────────────────────────┐
│ src/                    │       │ drafts/                  │
│ package.json            │       │   HTGO2-123/             │
│ ...                     │  git  │     state.yml            │
│ specs/ ──────────────── ┼ ───── │     clarify-log.md       │
│   (submodule pointer)   │       │     prd.md               │
│ .specrity.yml       │       │ features/                │
└─────────────────────────┘       │   HTGO2-123/spec/prd.md  │
                                  └──────────────────────────┘
```

### Draft-First 狀態機

PM 經常同時處理多個 ticket，頻繁切 branch 很痛苦。
Draft-First：所有草稿都在 `drafts/`（同一個 branch），不需切 branch：

```
/specrity.pm.specify HTGO2-123  → drafts/HTGO2-123/ 寫入
/specrity.pm.specify HTGO2-456  → drafts/HTGO2-456/ 寫入（不影響 123）
/specrity.pm.specify HTGO2-123  → 偵測 drafts/HTGO2-123/ 存在 → Resume

定稿後 → 才建 feature branch → 搬到 features/ → push
```

### 跨 Session 持久化

AI Agent 在不同對話間完全沒有記憶，所有狀態持久化到檔案：

```
drafts/HTGO2-123/
├── state.yml          ← 做到哪了（phase, 第幾輪 clarify）
├── jira-snapshot.md   ← Jira 原始需求（偵測變更用）
├── clarify-log.md     ← 問答紀錄（跨 Session 關鍵）
└── prd.md             ← PRD 草稿
```

新對話 `/specrity.pm.specify HTGO2-123` → 讀取這些檔案 → 從上次中斷處繼續。

---

## 工作流詳細說明

### 🎯 PM 工作流

```
/specrity.pm.specify HTGO2-123
  → AI 從 Jira 拉取 Ticket 需求
  → 開始 Clarify 互動（問答持久化到 drafts/）
  → PM 可隨時離開，下次同一指令自動 Resume
  → 定稿後自動建 Feature Branch & Push

中途切換任務？
  /specrity.pm.specify HTGO2-456  ← 開始另一個
  /specrity.pm.specify HTGO2-123  ← 回來繼續
```

### 🔧 Engineer 工作流

```
/specrity.dev.plan HTGO2-123       → 讀 PRD → 分析程式碼 → 產出 plan.md
/specrity.dev.tasks HTGO2-123      → 拆解任務 → 產出 tasks.md → 建 Jira Sub-tasks
/specrity.dev.implement HTGO2-123  → 逐步實作 → 更新進度 → 更新 Jira 狀態
```

### ⚠️ Jira 連線失敗時

所有指令都內建降級處理：
- AI 提示手動貼上 Jira 內容
- 流程照常，跳過 Jira 自動更新
- 事後手動補上 Jira 狀態

---

## 設定參考

### `.specrity.yml`（commit 到 repo）

```yaml
spec_mode: submodule          # local | submodule | external
spec_path: specs/             # spec 目錄相對路徑
# jira_cloud_id: xxx          # 選填，MCP 自動偵測
# jira_project_key: PROJ      # 選填，從 ticket ID 解析
```

### 設定優先權

```
.specrity.yml  →  .env  →  自動偵測
  (commit 到 repo)    (個人)    (MCP / ticket ID)
```

---

## 支援的 AI 工具

| Agent Type | Tool Name | Target Path |
|---|---|---|
| `claude` | Claude Code (CLI) | `CLAUDE.md` |
| `gemini` | Gemini CLI | `GEMINI.md` |
| `copilot` | GitHub Copilot | `.github/agents/copilot-instructions.md` |
| `cursor-agent` | Cursor IDE | `.cursor/rules/specrity.mdc` |
| `windsurf` | Windsurf | `.windsurf/rules/specrity.md` |
| `agy` | Antigravity | `.agents/workflows/*.md` |
| `qwen` | Qwen Code | `QWEN.md` |
| `opencode` | opencode | `AGENTS.md` |
| `codex` | Codex CLI | `AGENTS.md` |
| `kilocode` | Kilo Code | `.kilocode/rules/specrity.md` |
| `auggie` | Auggie CLI | `.augment/rules/specrity.md` |
| `roo` | Roo Code | `.roo/rules/specrity.md` |
| `codebuddy` | CodeBuddy CLI | `CODEBUDDY.md` |
| `amp` | Amp | `AGENTS.md` |
| `shai` | SHAI | `SHAI.md` |
| `q` | Amazon Q CLI | `AGENTS.md` |
| `bob` | IBM Bob | `AGENTS.md` |

## License

MIT
