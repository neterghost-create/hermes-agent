# 實戰範例：HRM-Enhanced 協議測試 (2026-06-09)

## 任務：寫日誌分析腳本 (log-analyzer.py)

### 觀察緩衝區 (低層步驟)

| # | 類型 | 信號 |
|---|------|------|
| 1 | task_start | 估計 5+ tool calls |
| 2 | observation | syslog 5655 行可用 |
| 3 | tool_failure | 正則全 miss，229 條 unknown |
| 4 | debug | 根因：ISO 8601 微秒精度 `.482562` |
| 5 | fix | 更新正則 `(?:\.\d+)?` |
| 6 | verify | 修復成功，6 個服務正確識別 |

### ACT 判斷

1 次 tool failure + 修復成功 → M=2 (兩輪反思)

### 反思週期 #1

```yaml
correction_pattern:
  trigger: "用正則解析 ISO 8601 時間戳"
  wrong: "只匹配 HH:MM:SS，沒有處理微秒部分"
  right: "ISO 8601 可能有微秒 (.NNNNNN)，正則要加 (?:\\.\\d+)?"
  check: "測試包含微秒的時間戳字符串"
  action: [discard]  # 太專門，不值得存
```

### 層級收斂

觀察緩衝區已清空，pattern 提取完成。

### 關鍵驗證點

- ✅ 觀察層正確捕獲了 tool failure 信號
- ✅ ACT 正確判斷了反思深度 (1 failure → M=2)
- ✅ 1-step gradient 生效：只記 pattern，不記流水帳
- ✅ 存儲決策正確：太專門的 pattern → discard
- ✅ 層級收斂：觀察緩衝區清空後進入下一週期

### ISO 8601 微秒陷阱 (附帶收穫)

```
# 錯誤: 只匹配到秒數
r"(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}[+\-]\d{4})"

# 正確: 包含可選微秒 + 帶冒號的時區偏移
r"(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?[+\-]\d{2}:?\d{2})"
```

syslog (rsyslog) 預設用 ISO 8601 with microseconds。journalctl 預設用 syslog 格式。正則要同時處理兩種。
