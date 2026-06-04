---
name: notebooklm-cli
description: 用 nlm CLI 直接操控 NotebookLM（建 notebook、上傳來源、取逐字稿、生成成品）。適用所有有 shell 的 agent，不依賴 MCP 是否載入。說「用 NotebookLM」「轉逐字稿」「上傳到 NotebookLM」時載入。
version: v0.3
updated: 2026-06-04
---

# 用 NotebookLM（nlm CLI，通用版）

> 核心觀念：**NotebookLM 沒有官方 API，但 `nlm` 這支 CLI 就是翻譯官。**
> MCP 只是 `nlm` 的包裝層。只要 `nlm` 在 PATH，**任何有 shell 的 agent 都能直接呼叫 `nlm`**，
> 不用管 MCP 有沒有載入。這是「所有 agent 都能用」的通用解。

## ⚠️ 為什麼直接用 CLI（實作教訓）

- 新版 **Cowork / Antigravity 桌面 app 不會載入 `~/.claude/settings.json` 的 `mcpServers`**，
  所以就算 `nlm setup add claude-code` 寫好設定、重啟，agent 裡也看不到 NotebookLM 工具。
- 解法：**別等 MCP，直接用 Bash 跑 `nlm`**。功能完全一樣。
- 繁體中文 Windows 跑 `nlm` 可能因 cp950 編碼 `UnicodeDecodeError` 崩潰
  → **每次都先 `export PYTHONUTF8=1 PYTHONIOENCODING=utf-8`**。

## 先備條件（環境檢查 / 自動安裝）

```bash
nlm --version            # 沒有 → uv tool install notebooklm-mcp-cli
nlm doctor               # 看登入狀態；未登入 → nlm login（開瀏覽器 Google 登入，一次即可）
```

- 沒有 `uv`：Windows `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`；
  macOS/Linux `curl -LsSf https://astral.sh/uv/install.sh | sh`
- `nlm: command not found` → 重開終端機；仍失敗把 `uv tool dir --bin` 的路徑加進 PATH。

## 核心管線（最常用）

```bash
export PYTHONUTF8=1 PYTHONIOENCODING=utf-8

# 1) 建 notebook（回傳 notebook-id）
nlm notebook create "我的主題"

# 2) 加來源並等處理完成（--wait）。檔案/URL/YouTube/文字皆可
nlm source add <notebook-id> --file "C:\path\audio.mp3" --wait --wait-timeout 540
nlm source add <notebook-id> --url https://example.com --wait
nlm source add <notebook-id> --youtube https://youtu.be/xxxx --wait
# 長音檔把 --wait-timeout 調大（秒）

# 3) 取內容
nlm source content <source-id> -o out.txt   # 原始逐字稿/文字（NotebookLM 會自動轉錄音檔）
nlm notebook query <notebook-id> "請用繁體中文條列重點"   # 問答（有依據、附引用）
```

## 範例：音檔 → 繁體中文逐字稿

```bash
export PYTHONUTF8=1 PYTHONIOENCODING=utf-8
NB=$(nlm notebook create "逐字稿" | grep -o '[0-9a-f-]\{36\}')
nlm source add "$NB" --file "audio.mp3" --title "音檔" --wait --wait-timeout 600
SRC=$(nlm source list "$NB" | grep -o '[0-9a-f-]\{36\}' | head -1)
nlm source content "$SRC" -o raw.txt
tr -d ' ' < raw.txt > clean.txt    # NotebookLM 輸出常每字帶空格，清掉
```

> ⚠️ 逐字稿是語音辨識（ASR），會有同音錯字（例：點商→點傷、預照→預兆）。
> 沉澱進筆記時要標註不確定、不要假裝是官方譯名。NotebookLM 對中文音檔多半直接輸出繁體。

## 生成成品（成品可下載 / 匯出）

```bash
nlm slides create <notebook-id>        # 簡報（可匯出 pptx）
nlm report create <notebook-id>        # 報告
nlm audio create <notebook-id>         # 音訊概覽（Podcast）
nlm video create <notebook-id>         # 影片概覽
nlm infographic create <notebook-id>   # 資訊圖表
nlm mindmap create <notebook-id>       # 心智圖
nlm quiz create <notebook-id>          # 測驗
nlm download <artifact-id> -o <資料夾> # 下載音/影片等成品
nlm export <artifact-id>               # 匯出到 Google Docs/Sheets
```

其他：`nlm list notebooks`、`nlm notebook list`、`nlm get`、`nlm describe`、`nlm research`、
`nlm batch`、`nlm cross`、`nlm pipeline`。指令詳列：`nlm --help`、`nlm --ai`。

## 讓所有 agent 都能用（一次設定）

```bash
# 安裝 skill（讓各 agent 原生知道 nlm 工作流）
nlm skill install claude-code
nlm skill install codex
nlm skill install gemini-cli
nlm skill install antigravity
nlm skill install opencode
nlm skill install agents          # 通用（Gemini/Codex/其他）

# 註冊 MCP（給「會載入 MCP」的 agent；不會載入的就靠上面的 CLI）
nlm setup add codex
nlm setup add gemini-cli
nlm setup add antigravity
nlm setup add opencode
nlm setup list                    # 看各 client MCP 狀態
nlm skill list                    # 看各 agent skill 安裝狀態
```

> Claude Code（Cowork 桌面版）：`nlm setup add claude-code` 對它無效（不載入 settings.json mcpServers，
> 且找不到 `claude` CLI）→ 直接用 `nlm` CLI 即可，不用糾結 MCP。

## 回報格式

執行後回報：nlm 版本、登入帳號、做了哪些步驟、notebook/source ID、成品/逐字稿存放位置。
