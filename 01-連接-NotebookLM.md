# Codex 懶人包 #01：連接 Google NotebookLM

> 版本：v0.3（實作後更新，跨 agent 通用）
> 更新日期：2026-06-04

> 📌 **本懶人包可獨立執行**：會自動檢查並安裝所需工具。

---

## 🆕 v0.3 實作後重大更新（先讀這段）

實際在 Cowork / Antigravity 桌面版操作後得到的關鍵教訓：

1. **不一定要把 MCP 接進 agent。** `nlm` 是一支獨立 CLI，MCP 只是它的包裝層。
   只要 `nlm` 在 PATH，**任何有 shell 的 agent（Claude Code、Codex、Gemini、Antigravity、OpenCode…）
   都能直接呼叫 `nlm` 完成所有 NotebookLM 操作** —— 這才是「所有 agent 都能用」的通用解。
2. **新版 Cowork / Antigravity 桌面 app 不載入 `~/.claude/settings.json` 的 `mcpServers`**。
   所以 `nlm setup add claude-code` 寫好設定、重啟，agent 裡仍看不到工具。→ 別等 MCP，直接用 CLI。
3. **繁中 Windows 跑 `nlm` 可能 cp950 編碼崩潰** → 每次先
   `export PYTHONUTF8=1 PYTHONIOENCODING=utf-8`（PowerShell：`$env:PYTHONUTF8=1`）。

➡️ 「直接用 CLI 操控 NotebookLM」的完整指令與範例（含音檔轉逐字稿）見
**[`skills/01-notebooklm/SKILL.md`](skills/01-notebooklm/SKILL.md)**。
本檔以下為「把 MCP 註冊進各 agent」的選配步驟。

### 一次讓所有 agent 都能用

```bash
export PYTHONUTF8=1 PYTHONIOENCODING=utf-8
# 安裝 skill（各 agent 原生知道 nlm 工作流）
for t in claude-code codex gemini-cli antigravity opencode agents; do nlm skill install "$t"; done
# 註冊 MCP（給會載入 MCP 的 agent）
for c in codex gemini-cli antigravity opencode; do nlm setup add "$c"; done
nlm skill list && nlm setup list   # 驗證
```

---

## 這個懶人包會幫你做什麼？

讓 Codex CLI 能直接操控 NotebookLM：建 notebook、上傳資料來源、產生簡報/資訊圖表/音訊/影片/心智圖/測驗等，成品自動下載到本機資料夾。

---

## 原理

```
Codex CLI ←(MCP 協定)→ nlm（翻譯官）←(Google 登入)→ NotebookLM
```

`nlm` 是 stdio MCP server，Codex 透過 MCP 協定呼叫它，它再去模擬瀏覽器操作 NotebookLM。

> 💡 **跟 Claude Code 版的差別**：MCP server 本體（`notebooklm-mcp-cli`）一模一樣，只是註冊到 Codex 的方式不同（Codex 用 `~/.codex/config.toml`，Claude Code 用 `~/.claude/settings.json`）。

---

## 先備條件

- [ ] Codex CLI 已安裝
- [ ] Google 帳號
- [ ] 電腦有網路連線

---

## 請 Codex 幫我執行以下步驟

### 步驟零：環境檢查

1. 作業系統
2. `git --version`，沒有就裝
3. `uv --version`，沒有則步驟一安裝
4. 網路連線
5. `codex --version`

---

### 步驟一：安裝 uv（如果沒裝）

**Windows（PowerShell）**：
```bash
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**macOS / Linux**：
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

### 步驟二：安裝 NotebookLM MCP CLI

```bash
uv tool install notebooklm-mcp-cli
```

確認：
```bash
nlm --version
```

> `nlm: command not found` → 重開終端機；仍失敗代表 uv 工具路徑沒進 PATH，告訴使用者把 `uv tool dir --bin` 印出的路徑加進 PATH。

---

### 步驟三：登入 Google 帳號

```bash
nlm login
```

> 🖐️ 瀏覽器會開 Google 登入頁，登入後 CLI 自動擷取認證。

確認：
```bash
nlm doctor
```

---

### 步驟四：把 NotebookLM 註冊為 Codex 的 MCP server

> ✅ 三種 Codex 共用 `~/.codex/config.toml`，做一次三邊都吃到。任選一條路：

**方法 A：Codex Desktop GUI（最推薦）**

1. 開 Codex Desktop → 設定 → **Integrations & MCP**
2. 點 **Add server**（或類似的「新增」按鈕）
3. 填：
   - Name：`notebooklm`
   - Command：`nlm`
   - Args：`mcp`
4. 儲存

**方法 B：手動編輯 `~/.codex/config.toml`**

> Desktop：設定 → Integrations & MCP → 「Open config.toml」也能直接打開這檔；IDE：齒輪 → MCP settings → Open config.toml；用記事本/編輯器開亦可。

```toml
[mcp_servers.notebooklm]
command = "nlm"
args = ["mcp"]
```

> 💡 路徑：Windows `C:\Users\<你>\.codex\config.toml`，macOS/Linux `~/.codex/config.toml`。檔案不存在就新建。
>
> ⚠️ **Section 名稱必須是 `mcp_servers`**（底線、複數）。寫成 `mcp-servers` 或 `mcpservers` Codex 會靜默忽略。

**方法 C：CLI（只給有裝 CLI 的人）**

```bash
codex mcp add notebooklm -- nlm mcp
```

---

### 步驟五：建立本地資料夾

在 Documents 下建：
```
Documents/
  └── NotebookLM/
      ├── slides/
      ├── infographics/
      ├── audio/
      ├── video/
      ├── docs/
      ├── sheets/
      ├── mindmaps/
      └── quizzes/
```

---

### 步驟六：重啟 Codex 並驗證

> 🖐️ **Desktop**：完全結束 app（不只是關視窗），重新開啟。
> **IDE 擴充**：Reload Window（VSCode：`Cmd/Ctrl+Shift+P` → Reload Window）。
> **CLI**：`exit` 後重新 `codex`。

驗證：
1. 對 Codex 說「列出我的 NotebookLM 筆記本清單」
2. 能成功列出（即使空的）→ 連接成功
3. Desktop 用戶可在 Integrations & MCP 設定面板看 `notebooklm` 顯示為 ✅ 連線中

---

### 步驟七：功能測試

1. 建一個叫「測試筆記本」的 notebook
2. 確認建立成功
3. 刪除它
4. ✅ 「全部完成！Codex 已連接 NotebookLM。」

---

## 如果失敗

對 Codex 說：「NotebookLM 懶人包執行失敗，清除設定重跑。」

復原：
- **Desktop**：設定 → Integrations & MCP → 找到 `notebooklm` → 刪除/停用
- **手動**：編輯 `~/.codex/config.toml` 移除 `[mcp_servers.notebooklm]` 段
- **CLI**：`codex mcp remove notebooklm`

清掉 nlm 本體：
```bash
uv tool uninstall notebooklm-mcp-cli
nlm logout 2>/dev/null
```

---

## 常見問題

| 問題 | 解法 |
|------|------|
| `nlm: command not found` | 重開終端機；把 `uv tool dir --bin` 路徑加進 PATH |
| 登入後 `nlm doctor` 顯示未認證 | 重跑 `nlm login` |
| Codex 看不到 NotebookLM 工具 | Desktop：設定 → Integrations & MCP 看 `notebooklm` 是否啟用 / 連線；CLI：`codex mcp list` |
| `codex mcp` 指令找不到 | 你沒裝 CLI，用 Desktop GUI 或手動編輯 config.toml |
| 設定改了沒生效 | section 名稱要 `mcp_servers`（底線、複數），寫錯會被靜默忽略 |
| Windows 上指令格式錯誤 | 用 PowerShell 或 Git Bash，別用 CMD |

---

## 相關連結

- [notebooklm-mcp-cli GitHub](https://github.com/jacob-bd/notebooklm-mcp-cli)
- [Codex MCP 官方文件](https://developers.openai.com/codex/mcp)
