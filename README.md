# n8n YouTube 自動回覆留言 + LINE 通知

用 n8n 自動監控 YouTube 頻道留言，回覆含關鍵字的留言，並推播 LINE 通知。

## 線上教學文件

👉 **[點此查看完整教學（Windows / Mac）](https://kuan35.github.io/n8n-tutorial/)**

## Workflow 說明

| 檔案 | 說明 |
|---|---|
| `wf_ytconfig_detail.json` | **yt答覆設定**：LINE 傳訊息設定監控規則，AI 解析後寫入 Google Sheets |
| `wf_v2_current.json` | **YT自動回覆留言 v2**：每 5 分鐘掃描新留言、自動回覆、推播 LINE 通知 |

## 匯入方式

1. 開啟 n8n → 左側 **Workflows** → 右上角選單 → **Import from file**
2. 選擇上方其中一個 `.json` 檔案
3. 依照[教學文件](https://kuan35.github.io/n8n-tutorial/)補上自己的憑證

## 教學文件

- [Windows 版教學](教學文件.md)
- [Mac 版教學](教學文件-mac.md)
