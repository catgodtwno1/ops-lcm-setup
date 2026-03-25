# ops-lcm-fix

OpenClaw LCM（Lossless-Claw Memory）壓縮故障診斷與修復技能。

## 功能

- **診斷 LCM 壓縮失敗** — 區分真實認證錯誤 vs 假陽性
- **修復假陽性 Auth Error** — 摘要內容包含 "401"、"unauthorized" 等關鍵字導致誤判
- **MiniMax apiKey 問題繞行** — 避免 env-marker 字面量導致的真實 401
- **完整診斷流程** — 從日誌檢查到 patch 應用到驗證

## 適用場景

- LCM 壓縮靜默失敗，上下文無限增長
- Gateway 日誌顯示 `provider auth error` 但模型實際可用
- 使用 MiniMax 作為 summaryModel 時出現 401
- 需要確認 LCM 插件是否正常工作

## 已知 Bug

### Bug 1：摘要內容觸發假陽性 Auth Error（v0.5.1）

**上游 Issue：** [Martian-Engineering/lossless-claw#171](https://github.com/Martian-Engineering/lossless-claw/issues/171)

當被壓縮的對話內容涉及認證錯誤（例如討論 "401 unauthorized"），摘要文本中的這些關鍵字會被 `AUTH_ERROR_TEXT_PATTERN` 正則匹配，導致成功的壓縮被誤判為認證失敗。

**修復：** 修改 `pickAuthInspectionValue()` 函數，當沒有找到錯誤相關欄位時返回空物件 `{}` 而非完整回應。

### Bug 2：MiniMax apiKey 環境變量標記問題

`models.json` 中的 apiKey 被存為字面量字串 `"MINIMAX_API_KEY"` 而非實際密鑰。每次 gateway 重啟都會重新生成此檔案。

**繞行方案：** summaryModel 使用 Anthropic 模型（如 `claude-haiku-4-5`）。

## 推薦配置

```json
{
  "summaryModel": "anthropic/claude-haiku-4-5",
  "threshold": 0.75
}
```

## 安裝

將此目錄放到 `~/.openclaw/workspace/skills/` 下，OpenClaw 會自動載入。

## 授權

MIT
