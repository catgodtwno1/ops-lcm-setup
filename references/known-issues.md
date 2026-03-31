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

### 5.5 CJK 搜索（lcm_grep）召回率低

**症狀**：`lcm_grep` 搜中文關鍵詞返回空結果或命中極少，但 summary 確實包含相關內容。

**根因**：
- FTS5 默認使用 porter tokenizer，對 CJK 文字分詞效果差
- 原始搜索邏輯用 AND 語義連接 bigram tokens，導致只要一個 bigram 不命中就全部失敗
- 沒有 trigram 索引表做子字串匹配

**修復**（已套用在 Scott#1 的 `~/.openclaw/extensions/lossless-claw/`）：

#### 修改 1：migration.ts — 新增 CJK trigram FTS 虛擬表

位置：`src/db/migration.ts`

```sql
CREATE VIRTUAL TABLE summaries_fts_cjk USING fts5(
  summary_id UNINDEXED,
  content,
  tokenize='trigram'
);
-- 回填既有 summaries
INSERT INTO summaries_fts_cjk(summary_id, content)
  SELECT summary_id, content FROM summaries;
```

`trigram` tokenizer 對 CJK 做 3-char 滑動窗口索引，支援任意子字串 MATCH。

#### 修改 2：summary-store.ts — 雙層搜索 + OR 語義

位置：`src/store/summary-store.ts`

**寫入時**同步插入 trigram 表：
```typescript
// summary 寫入後追加
this.db
  .prepare(`INSERT INTO summaries_fts_cjk(summary_id, content) VALUES (?, ?)`)
  .run(input.summaryId, input.content);
```

**搜索時**先走 trigram 表，再 fallback 到 LIKE OR：

1. **searchCjkTrigram()**：
   - 提取 CJK 段落（≥3 字元），拆成 4-char + 3-char 滑動窗口
   - 各 chunk 用 FTS5 MATCH 搜索，**OR 語義**合併
   - 非 CJK tokens（英文/數字）同時搜 porter FTS 表，合併結果

2. **searchLikeCjk()**（fallback）：
   - 提取 CJK bigrams（2-char 滑動窗口）
   - 每個 bigram 用 `LIKE '%xx%'` 搜索，**OR 語義**（任一命中即返回）
   - 結果按 created_at DESC 排序

**關鍵改動**：原始碼搜 CJK 走 `searchLike()`，用 AND 語義；修改後改為 OR 語義（trigram 優先 → LIKE OR fallback）。

#### 驗證

```bash
# 確認 trigram 表存在
sqlite3 ~/.openclaw/lcm.db "SELECT COUNT(*) FROM summaries_fts_cjk;"

# 測試 CJK 搜索
# 在 OpenClaw 會話中：
# lcm_grep(pattern="彩尚SEO", mode="full_text")
# 應返回相關 summary snippets
```

**狀態**：已在 Scott#1 本地套用，未提交上游 PR。修改在 `~/.openclaw/extensions/lossless-claw/src/` 的 `summary-store.ts` 和 `db/migration.ts`。
備份：`summary-store.ts.bak`（原始版本）。

