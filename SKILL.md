---
name: ops-lcm-setup
description: LCM (Lossless-Claw) 完整指南：安裝、配置、混搭模型省成本、假陽性 auth error 修復、MiniMax apiKey env-marker 問題。新機器部署 / LCM 故障排查 / 壓縮不觸發時載入。
---

# LCM (Lossless Context Management) 安裝與配置指南

> 基於 Scott#4 + Scott#2 實戰部署經驗整理。
> 上游倉庫：[Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw)
> Fork（含修復）：[catgodtwno4/lossless-claw](https://github.com/catgodtwno4/lossless-claw)

## 一、什麼是 LCM

LCM 取代 OpenClaw 內建的滑動窗口壓縮，改用 DAG 式摘要系統：
- **持久化每條訊息** → SQLite 儲存
- **階層式摘要** → 多層壓縮，保留關鍵資訊
- **智慧重建** → 每次對話從 DAG 挑選最相關節點
- **零遺失** → 原始訊息永遠保留，可展開還原

## 二、安裝步驟

### 2.1 安裝插件

```bash
# 方法 1：CLI 安裝（推薦）
openclaw plugins install lossless-claw

# 方法 2：TUI 安裝
openclaw tui
# 在 TUI 中輸入：/plugins install lossless-claw

# 方法 3：手動安裝
cd ~/.openclaw/extensions
npm install @martian-engineering/lossless-claw
```

### 2.2 啟用插件

```bash
openclaw plugins enable lossless-claw
```

### 2.3 設為上下文引擎（關鍵！）

在 `openclaw.json` 中加入：

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    }
  }
}
```

> ⚠️ **必須設 `contextEngine`**，否則 LCM 雖然 loaded 但不會接管壓縮。
> 這是 Scott#2 首次部署時踩的坑 — 插件載入了但沒有 wired。

### 2.4 驗證安裝

```bash
# 確認插件已載入
openclaw plugins list | grep lossless-claw
# 應顯示：lossless-claw  loaded  v0.5.1

# 確認 contextEngine 已設定
openclaw config get plugins.slots.contextEngine
# 應顯示：lossless-claw

# 確認 Gateway 識別
openclaw doctor --quick
```

## 三、混搭模型配置（省成本）

### 3.1 原理

主模型（Claude Opus）處理對話，LCM 用便宜模型（MiniMax M2.7 HS）生成摘要。
成本差異：Opus $15/M vs M2.7 HS ~$0.7/M — 約 20 倍差價。

### 3.2 配置範例

```json
{
  "models": {
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimaxi.com/anthropic",
        "apiKey": "${MINIMAX_API_KEY}",
        "headers": {
          "anthropic-version": "2023-06-01"
        }
      }
    }
  },
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    },
    "entries": {
      "lossless-claw": {
        "config": {
          "summaryModel": "minimax/MiniMax-M2.7-highspeed",
          "summaryProvider": "minimax",
          "expansionModel": "minimax/MiniMax-M2.7-highspeed",
          "expansionProvider": "minimax",
          "contextThreshold": 0.75
        }
      }
    }
  }
}
```

### 3.3 launchd 環境變數（macOS 必做）

> ⚠️ **踩坑重點**：LCM 的 `getApiKey()` 走 `process.env`，不走 `models.providers.*.apiKey`。
> macOS launchd 環境下，必須在 plist 裡設環境變數。

```bash
# 找到 plist
ls ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 編輯，在 <dict> 的 EnvironmentVariables 區段加入：
# <key>MINIMAX_API_KEY</key>
# <string>你的 MiniMax API Key</string>

# 或用 openclaw 內建（如果支援）
openclaw config set env.MINIMAX_API_KEY "sk-cp-xxx"

# 重啟 Gateway 生效
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

### 3.4 驗證混搭生效

```bash
# 查看最新 summary 使用的模型
sqlite3 ~/.openclaw/lcm.db \
  "SELECT summary_id, model, created_at FROM summaries ORDER BY created_at DESC LIMIT 5;"

# 預期輸出：model 欄位應為 MiniMax-M2.7-highspeed
```

## 四、配置參數說明

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `contextThreshold` | `0.75` | 觸發壓縮的 context 使用率（0.1=10% 即觸發，用於測試） |
| `summaryModel` | 主模型 | 生成摘要的模型 |
| `summaryProvider` | 主 provider | 摘要模型的 provider |
| `expansionModel` | 主模型 | 展開摘要的模型 |
| `expansionProvider` | 主 provider | 展開模型的 provider |
| `incrementalMaxDepth` | `5` | DAG 最大摘要深度 |
| `freshTailCount` | `10` | 保留最近 N 條原始訊息不壓縮 |
| `dbPath` | `~/.openclaw/lcm.db` | SQLite 資料庫路徑 |

### 生產建議值

```json
{
  "contextThreshold": 0.75,
  "summaryModel": "minimax/MiniMax-M2.7-highspeed",
  "summaryProvider": "minimax",
  "expansionModel": "minimax/MiniMax-M2.7-highspeed",
  "expansionProvider": "minimax"
}
```

### 測試用加速值

```json
{
  "contextThreshold": 0.1
}
```

> 測試完畢後記得改回 0.75！

## 五、已知問題與解法

See [references/known-issues.md](references/known-issues.md) for false-positive auth errors, MiniMax apiKey env-marker issues, compaction not triggering, schema migration, and other known issues.

## 六、部署 Checklist

新機器部署 LCM 時，逐項確認：

- [ ] `openclaw plugins install lossless-claw` 完成
- [ ] `openclaw plugins list` 顯示 lossless-claw loaded
- [ ] `plugins.slots.contextEngine: "lossless-claw"` 已設
- [ ] `models.providers.<provider>.apiKey` 有值（混搭時）
- [ ] launchd plist EnvironmentVariables 有對應 env var（macOS）
- [ ] `contextThreshold` 設為 0.75（生產）或 0.1（測試）
- [ ] Gateway 已重啟：`launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway`
- [ ] 驗證：對話 3-5 輪後查 `sqlite3 ~/.openclaw/lcm.db "SELECT * FROM summaries ORDER BY created_at DESC LIMIT 3;"`
- [ ] 驗證：model 欄位顯示正確的摘要模型（如 MiniMax-M2.7-highspeed）
- [ ] 驗證：err log 無 401 auth error

## 七、實測數據

| 機器 | 主模型 | LCM 模型 | Summaries | 漂移 | Errors |
|------|--------|---------|-----------|------|--------|
| Scott#1 | Claude | Haiku 4.5 | 正常 | — | Bug 1 假陽性（已 patch） |
| Scott#2 | Claude Opus | M2.7 HS | 40+ | 零 | 零（修復後） |
| Scott#4 | Claude Opus | M2.7 HS | 200+ | 零 | 零 |

## 八、相關資源

| 資源 | 連結 |
|------|------|
| 上游倉庫 | [Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw) |
| Fork（含修復） | [catgodtwno4/lossless-claw](https://github.com/catgodtwno4/lossless-claw) |
| LCM 論文 | [Voltropy LCM Paper](https://papers.voltropy.com/LCM) |
| 視覺化展示 | [losslesscontext.ai](https://losslesscontext.ai) |
| 假陽性修復 PR | [#178](https://github.com/Martian-Engineering/lossless-claw/pull/178) |
| getApiKey 修復 PR | [#179](https://github.com/Martian-Engineering/lossless-claw/pull/179) |
| Scott#1 診斷記錄 | [catgodtwno1/ops-lcm-fix](https://github.com/catgodtwno1/ops-lcm-fix) |

<!-- 在此追加新的部署經驗 -->
