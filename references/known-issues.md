## 五、已知問題與解法

### 5.1 假陽性 Auth Error

**症狀**：log 報 `compaction failed: provider auth error`，但 DB 裡 summary 已成功寫入。

**根因**：`summarize.ts` 的 `pickAuthInspectionValue()` 將對話內容中的 "401"/"authentication_error" 字串誤判為真正的 auth failure。

**解法**：
- 已提交 PR：[Martian-Engineering/lossless-claw#178](https://github.com/Martian-Engineering/lossless-claw/pull/178)
- 本地修復：
```bash
sed -i '' 's/Object.keys(subset).length > 0 ? subset : value/Object.keys(subset).length > 0 ? subset : {}/' \
  ~/.openclaw/extensions/lossless-claw/src/summarize.ts
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

### 5.2 混搭 Provider 401

**症狀**：主模型正常但 LCM 呼叫 401 invalid api key。

**根因有兩層**：

1. **`getApiKey()` 缺 config fallback**：fallback 鏈不包含 `models.providers.*.apiKey`，只有 `complete()` 函數有。
2. **`models.json` env-marker 覆蓋**（Scott#1 發現）：Gateway 重啟時自動重新生成 `~/.openclaw/models.json`，將 apiKey 寫為字面量字串 `"MINIMAX_API_KEY"`（env 變數名而非實際值）。手動修改會在下次重啟被覆蓋。

**如何判斷是哪個根因**：
```bash
# 檢查 models.json 裡的 apiKey 是字面量還是真值
grep -A1 'minimax' ~/.openclaw/models.json | grep apiKey
# 如果顯示 "MINIMAX_API_KEY"（沒有 sk- 前綴）→ Bug 2
# 如果顯示真實 key 或為空 → Bug 1
```

**解法（三選一，推薦方案 A）**：

- **方案 A（推薦）**：launchd plist 加 env var（見 3.3 節）→ `process.env.MINIMAX_API_KEY` 有值，繞過 models.json
- **方案 B**：v0.5.1 已含 `findProviderConfigValue` 修復 → 確保 `models.providers.minimax.apiKey` 在 openclaw.json 中有真值
- **方案 C**（僅 workaround）：summaryModel 改用 Anthropic（如 `claude-haiku-4-5`），避開 MiniMax

**相關 PR/Issue**：
- 已提交 PR：[Martian-Engineering/lossless-claw#179](https://github.com/Martian-Engineering/lossless-claw/pull/179)
- Scott#1 診斷記錄：[catgodtwno1/ops-lcm-fix](https://github.com/catgodtwno1/ops-lcm-fix)

### 5.3 Compaction 不觸發

**可能原因**：
1. 缺 `plugins.slots.contextEngine: "lossless-claw"` → 最常見
2. `contextThreshold` 太高，context 沒達到閾值
3. `compaction.mode: "safeguard"` 攔截了 CLI 注入的訊息

**排查步驟**：
```bash
# 1. 確認 contextEngine
grep -A2 '"contextEngine"' ~/.openclaw/openclaw.json

# 2. 查看 compaction log
grep -i 'compaction\|compact\|lcm' ~/.openclaw/gateway.err.log | tail -20

# 3. 查看 DB 狀態
sqlite3 ~/.openclaw/lcm.db "SELECT COUNT(*) FROM summaries;"
sqlite3 ~/.openclaw/lcm.db "SELECT COUNT(*) FROM messages;"
```

### 5.4 插件載入但不生效

**症狀**：`openclaw plugins list` 顯示 loaded，但 compaction 仍走內建邏輯。

**根因**：缺少 `plugins.slots.contextEngine` 設定。

**解法**：
```bash
# 加入 contextEngine slot
# 在 openclaw.json 的 plugins.slots 裡加：
# "contextEngine": "lossless-claw"
openclaw config set plugins.slots.contextEngine lossless-claw
```

