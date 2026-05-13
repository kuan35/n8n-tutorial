# n8n YouTube 自動回覆留言 + LINE 通知 完整教學

> 本教學實作兩個 n8n Workflow：
> 1. **yt答覆設定**：透過 LINE 訊息設定「哪支影片要監控、關鍵字是什麼、自動回覆什麼」
> 2. **YT自動回覆留言 v2**：每 5 分鐘自動掃描 YouTube 新留言、自動回覆、推播 LINE 通知

---

## 目錄

1. [系統需求](#1-系統需求)
2. [Docker 安裝 n8n](#2-docker-安裝-n8n)
3. [ngrok 固定網域設定](#3-ngrok-固定網域設定)
4. [重新啟動 n8n（含 ngrok 網域）](#4-重新啟動-n8n含-ngrok-網域)
5. [Google Cloud Console 設定](#5-google-cloud-console-設定)
6. [Google Gemini API 申請](#6-google-gemini-api-申請)
7. [LINE Messaging API 申請](#7-line-messaging-api-申請)
8. [n8n 建立四個憑證](#8-n8n-建立四個憑證)
9. [Google Sheets 建立試算表](#9-google-sheets-建立試算表)
10. [Workflow 1：yt答覆設定（5個節點）](#10-workflow-1yt答覆設定5個節點)
11. [Workflow 2：YT自動回覆留言 v2（12個節點）](#11-workflow-2yt自動回覆留言-v212個節點)
12. [LINE Webhook 設定](#12-line-webhook-設定)
13. [測試與驗證](#13-測試與驗證)
14. [常見錯誤排除](#14-常見錯誤排除)

---

## 1. 系統需求

| 項目 | 說明 |
|---|---|
| 作業系統 | Windows 10/11、macOS、Linux |
| Docker Desktop | 用於執行 n8n 容器 |
| ngrok 帳號 | 免費方案即可（提供一個固定 domain） |
| Google 帳號 | 用於 YouTube、Google Sheets、Gemini |
| LINE 帳號 | 用於 Messaging API 機器人 |

---

## 2. Docker 安裝 n8n

### 2-1 安裝 Docker Desktop

前往 [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) 下載並安裝，安裝完成後確認 Docker 正在執行（工具列有鯨魚圖示）。

### 2-2 建立 n8n 資料夾

在本機建立一個資料夾存放 n8n 設定，例如：
```
C:\n8n-data\
```

### 2-3 建立 docker-compose.yml

在上述資料夾內建立 `docker-compose.yml`，內容如下：

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=https://YOUR_NGROK_DOMAIN.ngrok-free.app
      - GENERIC_TIMEZONE=Asia/Taipei
      - N8N_SECURE_COOKIE=false
    volumes:
      - ./n8n-data:/home/node/.n8n
```

> **注意**：`WEBHOOK_URL` 先保持原樣，第 4 節取得 ngrok domain 後再回來修改。

### 2-4 第一次啟動 n8n

在終端機進入資料夾，執行：
```bash
docker compose up -d
```

等待約 30 秒後，開啟瀏覽器前往 [http://localhost:5678](http://localhost:5678)，依照畫面提示建立管理員帳號。

---

## 3. ngrok 固定網域設定

### 3-1 申請帳號並取得 Authtoken

1. 前往 [https://ngrok.com](https://ngrok.com) 註冊免費帳號
2. 登入後前往 **Your Authtoken** 頁面，複製 Token
3. 下載並安裝 ngrok（或使用 `choco install ngrok`）

### 3-2 設定 Authtoken

在終端機執行：
```bash
ngrok config add-authtoken YOUR_TOKEN_HERE
```

### 3-3 取得免費固定 Domain

1. 登入 ngrok Dashboard
2. 左側選單點選 **Domains**
3. 點選 **New Domain**，系統會分配一個固定 domain，例如：
   ```
   uncallused-semiexpressionistic-deane.ngrok-free.app
   ```
4. 記下這個 domain

### 3-4 啟動 ngrok 隧道

```bash
ngrok http --domain=YOUR_FIXED_DOMAIN.ngrok-free.app 5678
```

啟動後，ngrok 會顯示連線狀態。這個視窗必須保持開著，關掉就斷線。

---

## 4. 重新啟動 n8n（含 ngrok 網域）

回到 `docker-compose.yml`，將 `WEBHOOK_URL` 改為你的 ngrok domain：

```yaml
- WEBHOOK_URL=https://YOUR_FIXED_DOMAIN.ngrok-free.app
```

重啟 n8n：
```bash
docker compose down
docker compose up -d
```

確認 n8n 仍可透過 [http://localhost:5678](http://localhost:5678) 存取。

---

## 5. Google Cloud Console 設定

這個步驟需要啟用 **Google Sheets API**、**YouTube Data API v3**，並建立 OAuth 2.0 憑證，讓 n8n 能代表你操作 Google 服務。

### 5-1 建立 Google Cloud 專案

1. 前往 [https://console.cloud.google.com](https://console.cloud.google.com)
2. 點選頁面上方的專案下拉選單 → **新增專案**
3. 輸入專案名稱（例如 `n8n-youtube`）→ 建立

### 5-2 啟用 Google Sheets API

1. 左側選單 → **API 和服務** → **程式庫**
2. 搜尋 `Google Sheets API` → 點選 → **啟用**

### 5-3 啟用 YouTube Data API v3

1. 同上，搜尋 `YouTube Data API v3` → 點選 → **啟用**

### 5-4 建立 OAuth 同意畫面

1. 左側 → **API 和服務** → **OAuth 同意畫面**
2. 選擇 **外部** → **建立**
3. 填入應用程式名稱（例如 `n8n`）、你的 Email
4. 點選**儲存並繼續**（範圍頁面不需要修改）
5. **測試使用者**：點選 **Add Users**，加入你的 Google 帳號 Email
6. 完成

### 5-5 建立 OAuth 2.0 用戶端 ID

1. 左側 → **API 和服務** → **憑證**
2. 上方 → **建立憑證** → **OAuth 用戶端 ID**
3. 應用程式類型選 **網頁應用程式**
4. 名稱填 `n8n`
5. **已授權的重新導向 URI** → 新增：
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
6. 點選**建立**
7. 畫面會顯示「用戶端 ID」和「用戶端密鑰」→ **兩者都要複製保存**

---

## 6. Google Gemini API 申請

1. 前往 [https://aistudio.google.com](https://aistudio.google.com)
2. 點選左側 **Get API key**
3. 點選 **建立 API 金鑰**，選擇你的 Google Cloud 專案
4. 複製 API 金鑰並保存（格式類似 `AIzaSy...`）

---

## 7. LINE Messaging API 申請

### 7-1 建立 LINE Developer 帳號

1. 前往 [https://developers.line.biz](https://developers.line.biz)
2. 使用你的 LINE 帳號登入

### 7-2 建立 Provider 與 Channel

1. 點選 **Create a new provider**，輸入名稱（例如 `我的機器人`）
2. 選擇 **Messaging API** → **Create a Messaging API channel**
3. 填入：
   - Channel name：機器人名稱
   - Channel description：簡短說明
   - Category / Subcategory：隨意選擇
4. 同意條款 → **Create**

### 7-3 取得 Channel Access Token

1. 進入你剛建立的 Channel
2. 點選 **Messaging API** 標籤
3. 下拉到 **Channel access token** → 點選 **Issue**
4. 複製長串的 Token（格式：`...==`）

### 7-4 關閉自動回覆

1. 同一頁面，點選 **LINE Official Account Manager** 連結
2. 進入後點選 **設定** → **回應設定**
3. 關閉「自動回應訊息」、關閉「加入好友的歡迎訊息」（否則機器人會自動亂回）

### 7-5 取得你的 LINE User ID

你需要自己的 User ID 用於推播通知：
1. 在 LINE Developers → **Basic settings** 標籤
2. 下拉到 **Your user ID**，複製 `U` 開頭的 ID

### 7-6 加這個機器人為 LINE 好友

1. 在 **Messaging API** 標籤，有一個 QR Code
2. 用 LINE 掃描加好友（必須先加好友才能接收推播）

---

## 8. n8n 建立四個憑證

進入 n8n（[http://localhost:5678](http://localhost:5678)），左側選單 → **Credentials** → **Add Credential**

### 憑證 1：Google Sheets（OAuth2）

1. 搜尋 `Google Sheets OAuth2 API` → 選擇
2. 填入：
   - **Client ID**：Google Cloud Console 的用戶端 ID
   - **Client Secret**：Google Cloud Console 的用戶端密鑰
3. 點選 **Sign in with Google** → 授權你的 Google 帳號
4. 儲存，名稱設為 `Google Sheets account`

### 憑證 2：Google OAuth2（YouTube 用）

1. 搜尋 `Google OAuth2 API` → 選擇
2. 填入相同的 Client ID 和 Client Secret
3. **Scope** 欄位加入：
   ```
   https://www.googleapis.com/auth/youtube.force-ssl
   ```
4. 點選 **Sign in with Google** → 授權
5. 儲存，名稱設為 `Google OAuth2 account`

### 憑證 3：Google Gemini API

1. 搜尋 `Google Gemini(PaLM) API` → 選擇
2. 填入 **API Key**：貼上第 6 節取得的 Gemini API 金鑰
3. 儲存，名稱設為 `Google Gemini(PaLM) Api account`

### 憑證 4：LINE Header Auth

1. 搜尋 `Header Auth` → 選擇
2. 填入：
   - **Name**：`Authorization`
   - **Value**：`Bearer YOUR_LINE_CHANNEL_ACCESS_TOKEN`
   （把 `YOUR_LINE_CHANNEL_ACCESS_TOKEN` 換成第 7-3 節複製的 Token）
3. 儲存，名稱設為 `Header Auth account`

---

## 9. Google Sheets 建立試算表

### 9-1 建立試算表

1. 前往 Google Sheets，新增一個空白試算表
2. 命名為 `YT自動回覆留言`

### 9-2 建立欄位標頭

在第一列依序填入以下欄位名稱（**完全一致，含大小寫**）：

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| youtube_url | ID | 關鍵字 | 回應 | 檢查時間 | 已回覆留言ID |

### 9-3 記下試算表 ID

網址列格式為：
```
https://docs.google.com/spreadsheets/d/【這裡是試算表ID】/edit
```
複製並保存這個 ID，後續節點設定需要用到。

---

## 10. Workflow 1：yt答覆設定（5個節點）

**功能說明**：使用者在 LINE 傳送一段描述（包含影片網址、關鍵字、自動回覆文字），AI 解析後自動寫入 Google Sheets，並回傳設定確認訊息。

**節點流程**：
```
Webhook → AI Agent → Code in JavaScript → Append or update row in sheet → HTTP Request
                ↑
    Google Gemini Chat Model（AI Agent 的子節點）
```

### 10-1 建立新 Workflow

1. n8n 左側 → **Workflows** → **Add Workflow**
2. 將 Workflow 命名為 `yt答覆設定`

---

### 節點 1：Webhook

**作用**：接收 LINE 傳來的訊息。

1. 點選畫布空白處 → 搜尋 `Webhook` → 選擇
2. 設定：
   - **HTTP Method**：`POST`
   - **Path**：留空（n8n 會自動產生一個 UUID，之後複製這個 UUID 用於 LINE Webhook URL）
   - **Respond**：`Immediately`（不等待後續節點，這樣 LINE 不會 timeout）
3. 儲存節點

> **重要**：點選節點後，在 **Webhook URLs** 區塊複製 **Production URL**，格式為：
> ```
> https://YOUR_NGROK_DOMAIN/webhook/【UUID】
> ```
> 這個 URL 將在第 12 節貼到 LINE Developer Console。

---

### 節點 2：Google Gemini Chat Model（子節點）

**作用**：提供 AI Agent 使用的語言模型。

1. 在畫布下方，搜尋 `Google Gemini Chat Model` → 選擇
2. 設定：
   - **Credential**：選擇 `Google Gemini(PaLM) Api account`
   - **Model**：`gemini-2.0-flash`（或 `gemini-1.5-flash`）
3. **不要直接連接到流程**，這個節點要拖曳到 AI Agent 節點的 **Model** 連接埠

---

### 節點 3：AI Agent

**作用**：解析 LINE 使用者的訊息，提取影片網址、關鍵字、回覆文字，輸出結構化 JSON。

1. 搜尋 `AI Agent` → 選擇（位於 LangChain 分類）
2. 設定：
   - **Source**：`Define below`（自訂 Prompt）
   - **Prompt (User Message)**：
     ```
     ={{ $json.body.events[0].message.text }}
     ```
   - **System Message**：
     ```
     你是資料格式化助理。

     使用者會傳入包含網址、關鍵字和回覆文字的訊息。
     請解析後，只回傳以下格式的純文字，不要有任何其他說明：

     {"url":"完整網址","id":"影片ID","key":"關鍵字","reply":"回覆文字"}
     ```
3. 將 **Google Gemini Chat Model** 節點拖曳連接到 AI Agent 的 **Model** 插槽（節點左下角的 AI 圖示）
4. 將 **Webhook** 的輸出連接到 **AI Agent** 的輸入

---

### 節點 4：Code in JavaScript

**作用**：AI 有時會在 JSON 外面加上 ` ```json ``` ` 標籤，這個節點負責清除並解析成乾淨的 JSON 物件。

1. 搜尋 `Code` → 選擇
2. **Language**：`JavaScript`
3. 貼入以下程式碼：
   ```javascript
   const raw = $input.item.json.output.replace(/```json|```/g, '').trim();
   const data = JSON.parse(raw);
   return [{ json: data }];
   ```
4. 連接：**AI Agent** → **Code in JavaScript**

---

### 節點 5：Append or update row in sheet（Google Sheets）

**作用**：將解析後的資料新增一列到 Google Sheets。

1. 搜尋 `Google Sheets` → 選擇
2. 設定：
   - **Credential**：選擇 `Google Sheets account`
   - **Resource**：`Sheet`
   - **Operation**：`Append`（新增一列，不覆蓋）
   - **Document**：從下拉選單選擇你的試算表 `YT自動回覆留言`
   - **Sheet**：選擇 `工作表1`
3. **Columns** 區塊，切換到 **Map Each Column Manually**，設定以下對應：

   | 欄位名稱 | 值（Expression） |
   |---|---|
   | youtube_url | `{{ $json.url }}` |
   | ID | `{{ $json.id }}` |
   | 關鍵字 | `{{ $json.key }}` |
   | 回應 | `{{ $json.reply }}` |

4. 連接：**Code in JavaScript** → **Append or update row in sheet**

---

### 節點 6：HTTP Request（LINE 回覆）

**作用**：用 LINE Reply API 回傳設定確認訊息給使用者。

1. 搜尋 `HTTP Request` → 選擇
2. 設定：
   - **Method**：`POST`
   - **URL**：`https://api.line.me/v2/bot/message/reply`
   - **Authentication**：`Generic Credential Type`
   - **Generic Auth Type**：`Header Auth`
   - **Credential**：選擇 `Header Auth account`
3. 點選 **Add Header**，新增：
   - **Name**：`Content-Type`
   - **Value**：`application/json`
4. **Body** 區塊：
   - **Body Content Type**：`JSON`
   - **Specify Body**：`Using JSON`
   - 貼入以下內容（**整段貼入 JSON Body 欄位**）：
   ```
   ={{ JSON.stringify({
     replyToken: $('Webhook').item.json.body.events[0].replyToken,
     messages: [{
       type: 'text',
       text: '設定完成！\n\n影片：' + $('AI Agent').item.json.output.match(/"url":"([^"]+)"/)[1]
             + '\n關鍵字：' + $('AI Agent').item.json.output.match(/"key":"([^"]+)"/)[1]
             + '\n自動回覆：' + $('AI Agent').item.json.output.match(/"reply":"([^"]+)"/)[1]
     }]
   }) }}
   ```
5. 連接：**Append or update row in sheet** → **HTTP Request**

### Workflow 1 完成

點選右上角 **Publish** 啟用。

---

## 11. Workflow 2：YT自動回覆留言 v2（12個節點）

**功能說明**：每 5 分鐘自動讀取 Google Sheets 的規則清單，對每支影片抓取最新 YouTube 留言，自動回覆含關鍵字的留言，並推播 LINE 通知。

**節點流程**：
```
Schedule Trigger → 讀取規則 → Loop Over Items → GET YouTube留言 → 留言過濾
                                                                        ↓
                                                                    有動作?
                                                              False ↙      ↘ True
                                                           (skip)         需要回覆?
                                                                    True ↙    ↘ False
                                                               YouTube回覆      |
                                                                    ↓           |
                                                               更新已回覆紀錄 ←──┘
                                                                    ↓
                                                               AI生成通知 ← Google Gemini Chat Model
                                                                    ↓
                                                               LINE推播
                                                                    ↓
                                                               Loop（繼續下一列）
```

### 11-0 建立新 Workflow

1. n8n 左側 → **Workflows** → **Add Workflow**
2. 命名為 `YT自動回覆留言 v2`

---

### 節點 1：Schedule Trigger

**作用**：每 5 分鐘自動執行一次。

1. 搜尋 `Schedule Trigger` → 選擇
2. 設定：
   - **Trigger Interval**：`Minutes`
   - **Minutes Between Triggers**：`5`

---

### 節點 2：讀取規則（Google Sheets）

**作用**：讀取 Sheets 上所有影片規則列。

1. 搜尋 `Google Sheets` → 選擇
2. 設定：
   - **Credential**：選擇 `Google Sheets account`
   - **Resource**：`Sheet`
   - **Operation**：`Get Many Rows`
   - **Document**：選擇 `YT自動回覆留言`
   - **Sheet**：選擇 `工作表1`
   - **Return All**：開啟（讀取所有列）
3. 連接：**Schedule Trigger** → **讀取規則**

---

### 節點 3：Loop Over Items

**作用**：對每列規則逐一執行後續流程（一列一部影片）。

1. 搜尋 `Loop Over Items` → 選擇（或搜尋 `Split In Batches`）
2. 設定：
   - **Batch Size**：`1`（每次處理一列）
3. 連接：**讀取規則** → **Loop Over Items**（連接 Input 端口）

---

### 節點 4：GET YouTube留言（HTTP Request）

**作用**：呼叫 YouTube Data API 取得該影片最新留言。

1. 搜尋 `HTTP Request` → 選擇
2. 設定：
   - **Method**：`GET`
   - **URL**：
     ```
     https://www.googleapis.com/youtube/v3/commentThreads
     ```
   - **Authentication**：`Predefined Credential Type`
   - **Credential Type**：`Google OAuth2 API`
   - **Credential**：選擇 `Google OAuth2 account`
3. **Query Parameters** 新增以下參數：

   | Name | Value |
   |---|---|
   | part | `snippet` |
   | videoId | `={{ $('Loop Over Items').item.json['ID'] }}` |
   | maxResults | `20` |
   | order | `time` |

4. 連接：**Loop Over Items**（Done 端口）→ **GET YouTube留言**

---

### 節點 5：留言過濾（Code Node）

**作用**：核心邏輯。過濾已處理留言、找出關鍵字匹配的留言或新留言，決定動作（reply / notify / skip）。

1. 搜尋 `Code` → 選擇
2. **Language**：`JavaScript`
3. 貼入以下完整程式碼：

```javascript
const loopRow = $('Loop Over Items').item.json;
const ytItems = $input.first().json.items || [];

const keyword    = loopRow['關鍵字'] || '';
const replyText  = loopRow['回應'] || '';
const existingIds = loopRow['已回覆留言ID'] || '';
const ownChannelId = 'UCRc0GUnMU00Qiwc-aNWIwww';  // 換成你自己的頻道 ID

// 合併同影片所有列的已回覆ID，避免不同規則重複通知
const allRows = $('讀取規則').all().map(i => i.json);
const existingList = allRows
  .filter(r => r['youtube_url'] === loopRow['youtube_url'])
  .flatMap(r => (r['已回覆留言ID'] || '').split(',').filter(x => x.trim()));

// 過濾：排除已回覆 + 自己頻道
const newComments = ytItems.filter(item => {
  const id     = item.snippet.topLevelComment.id;
  const author = item.snippet.topLevelComment.snippet.authorChannelId?.value;
  return !existingList.includes(id) && author !== ownChannelId;
});

// 關鍵字 match 優先
const toReply  = newComments.filter(c =>
  c.snippet.topLevelComment.snippet.textDisplay.includes(keyword)
);
const toNotify = newComments.filter(c =>
  !c.snippet.topLevelComment.snippet.textDisplay.includes(keyword)
);

// 每次只處理第一則
const target = toReply[0] || toNotify[0] || null;
if (!target) return [{ json: { action: 'skip' } }];

const action = toReply.includes(target) ? 'reply' : 'notify';

// notify 只讓同影片 row_number 最小的列觸發，避免同一執行多列重複通知
if (action === 'notify') {
  const sameVideoMinRow = Math.min(...allRows
    .filter(r => r['youtube_url'] === loopRow['youtube_url'])
    .map(r => r['row_number']));
  if (loopRow['row_number'] !== sameVideoMinRow) {
    return [{ json: { action: 'skip' } }];
  }
}

const comment = target.snippet.topLevelComment;

return [{ json: {
  action,
  row_number:  loopRow['row_number'],
  commentId:   comment.id,
  text:        comment.snippet.textDisplay,
  author:      comment.snippet.authorDisplayName,
  videoId:     target.snippet.videoId,
  youtube_url: loopRow['youtube_url'],
  replyText,
  existingIds
}}];
```

> **注意**：第 5 行的 `UCRc0GUnMU00Qiwc-aNWIwww` 換成你自己的 YouTube 頻道 ID（在 YouTube Studio → 設定 → 頻道 → 基本資訊可找到）。

4. 連接：**GET YouTube留言** → **留言過濾**

---

### 節點 6：有動作?（If）

**作用**：判斷是否有需要處理的留言（非 skip）。

1. 搜尋 `If` → 選擇
2. **Conditions** 設定：
   - **Value 1**：`={{ $json.action }}`
   - **Operation**：`is not equal to`
   - **Value 2**：`skip`
3. **Options** → **Type Validation**：改為 `Loose`（避免空值報錯）
4. 連接：**留言過濾** → **有動作?**
   - **True** 端口 → 接下一個 If 節點
   - **False** 端口 → 接回 **Loop Over Items** 的 Loop 端口

---

### 節點 7：需要回覆?（If）

**作用**：判斷是否需要回覆 YouTube（含關鍵字）或只推通知。

1. 搜尋 `If` → 選擇
2. **Conditions** 設定：
   - **Value 1**：`={{ $json.action }}`
   - **Operation**：`equal to`
   - **Value 2**：`reply`
3. **Options** → **Type Validation**：`Loose`
4. 連接：**有動作?**（True）→ **需要回覆?**

---

### 節點 8：YouTube回覆（HTTP Request）

**作用**：在 YouTube 留言下方發布自動回覆。

1. 搜尋 `HTTP Request` → 選擇
2. 設定：
   - **Method**：`POST`
   - **URL**：`https://www.googleapis.com/youtube/v3/comments`
   - **Authentication**：`Predefined Credential Type`
   - **Credential Type**：`Google OAuth2 API`
   - **Credential**：選擇 `Google OAuth2 account`
3. **Query Parameters**：
   - `part`：`snippet`
4. **Body**：
   - **Body Content Type**：`JSON`
   - **Specify Body**：`Using JSON`
   - 貼入：
   ```
   ={{ JSON.stringify({
     snippet: {
       parentId: $('留言過濾').item.json.commentId,
       textOriginal: $('留言過濾').item.json.replyText
     }
   }) }}
   ```
5. 連接：**需要回覆?**（True）→ **YouTube回覆**

---

### 節點 9：更新已回覆紀錄（Google Sheets）

**作用**：將剛處理的留言 ID 存入 Sheets，避免下次重複處理。

1. 搜尋 `Google Sheets` → 選擇
2. 設定：
   - **Credential**：選擇 `Google Sheets account`
   - **Resource**：`Sheet`
   - **Operation**：`Update Row`
   - **Document**：選擇 `YT自動回覆留言`
   - **Sheet**：選擇 `工作表1`
3. **Columns** 設定（Map Each Column Manually）：

   | 欄位 | 值 |
   |---|---|
   | row_number | `={{ $('留言過濾').item.json['row_number'] }}` |
   | 已回覆留言ID | `={{ ($('留言過濾').item.json['existingIds'] ? $('留言過濾').item.json['existingIds'] + ',' : '') + $('留言過濾').item.json['commentId'] }}` |
   | 檢查時間 | `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}` |

4. **Matching Column**（用哪欄判斷要更新哪一列）：設為 `row_number`
5. 連接方式（兩條路都要接進來）：
   - **YouTube回覆** 的輸出 → **更新已回覆紀錄**
   - **需要回覆?**（False）→ **更新已回覆紀錄**

---

### 節點 10：Google Gemini Chat Model（子節點）

**作用**：AI生成通知的語言模型（與 Workflow 1 同一個設定）。

1. 搜尋 `Google Gemini Chat Model` → 選擇
2. **Credential**：選擇 `Google Gemini(PaLM) Api account`
3. **Model**：`gemini-2.0-flash`
4. 拖曳連接到下一個 AI Agent 節點的 **Model** 插槽

---

### 節點 11：AI生成通知（AI Agent）

**作用**：根據留言資訊，用 AI 生成一則繁體中文 LINE 通知訊息。

1. 搜尋 `AI Agent` → 選擇
2. 設定：
   - **Source**：`Define below`
   - **Prompt (User Message)**：
     ```
     ={{ $('留言過濾').item.json.action === 'reply'
       ? '已自動回覆 @' + $('留言過濾').item.json.author
         + ' 的留言：「' + $('留言過濾').item.json.text + '」'
         + '，回覆：' + $('留言過濾').item.json.replyText
       : '有新留言需確認，留言者：@' + $('留言過濾').item.json.author
         + '，內容：「' + $('留言過濾').item.json.text + '」'
     }}
     ```
   - **System Message**：
     ```
     你是 YouTube 頻道助理。根據以下資訊，用繁體中文生成一則簡短的 LINE 通知訊息。
     action=reply 時說已自動回覆；action=notify 時說有新留言待確認。
     要包含留言者名稱和留言內容。不要超過 3 行。不要加任何多餘說明。
     ```
3. 將 **Google Gemini Chat Model** 連接到此節點的 **Model** 插槽
4. 連接：**更新已回覆紀錄** → **AI生成通知**

---

### 節點 12：LINE推播（HTTP Request）

**作用**：將通知訊息推播給指定 LINE 使用者。

1. 搜尋 `HTTP Request` → 選擇
2. 設定：
   - **Method**：`POST`
   - **URL**：`https://api.line.me/v2/bot/message/push`
   - **Authentication**：`Generic Credential Type`
   - **Generic Auth Type**：`Header Auth`
   - **Credential**：選擇 `Header Auth account`
3. 點選 **Add Header**：
   - **Name**：`Content-Type`
   - **Value**：`application/json`
4. **Body**：
   - **Body Content Type**：`JSON`
   - **Specify Body**：`Using JSON`
   - 貼入（**把 `U265d12...` 換成你的 LINE User ID**）：
   ```
   ={{ JSON.stringify({
     to: 'U265d12aa422837205fe267ae054404f9',
     messages: [{
       type: 'text',
       text: $json.output + '\n\n影片連結：' + $('留言過濾').item.json.youtube_url
     }]
   }) }}
   ```
5. 連接：**AI生成通知** → **LINE推播**

---

### 最後一條連線：LINE推播 → Loop

將 **LINE推播** 的輸出連回 **Loop Over Items** 的 **Loop** 端口（左側那個，不是 Done），這樣才會繼續處理下一列規則。

### Workflow 2 完成

點選右上角 **Publish** 啟用。

---

## 12. LINE Webhook 設定

**目的**：讓 LINE 收到使用者訊息時，自動轉發到 n8n 的 yt答覆設定 Workflow。

1. 回到 [LINE Developers Console](https://developers.line.biz)
2. 進入你的 Messaging API Channel → **Messaging API** 標籤
3. 找到 **Webhook URL** 欄位 → 點選 **Edit**
4. 貼入 Workflow 1 Webhook 節點的 Production URL，格式：
   ```
   https://YOUR_NGROK_DOMAIN.ngrok-free.app/webhook/【Webhook UUID】
   ```
5. 點選 **Verify** 驗證連線（n8n 必須正在執行、Workflow 已 Publish）
6. 開啟 **Use webhook**（開關打開）

---

## 13. 測試與驗證

### 測試 yt答覆設定

1. 在 LINE 對機器人發送訊息，格式如：
   ```
   https://www.youtube.com/watch?v=影片ID 關鍵字：訂閱 回覆：感謝訂閱！
   ```
2. 機器人應在幾秒內回傳設定確認訊息
3. 確認 Google Sheets 多了一列資料

### 測試 YT自動回覆留言 v2

1. 用另一個帳號（非自己頻道）在監控的影片下留言，包含關鍵字
2. 在 n8n 手動點選 **Execute workflow** 或等待 5 分鐘
3. 確認：
   - YouTube 影片下出現自動回覆
   - LINE 收到通知（含影片連結）
   - Google Sheets 的 `已回覆留言ID` 欄被更新
   - 再等 5 分鐘，**不會**重複收到相同通知

---

## 14. 常見錯誤排除

### 問題 1：LINE Webhook 驗證失敗
- 確認 n8n 正在執行、ngrok 連線正常
- 確認 Workflow 1 已 **Publish**（啟用），不是 inactive 狀態
- 確認 Webhook URL 使用的是 Production URL（非 Test URL）

### 問題 2：LINE 回傳「JSON Body is not valid」
- 確認 LINE推播 節點的 JSON Body 用的是 `JSON.stringify(...)` 表達式
- 不要直接在 JSON 裡嵌入 `{{ $json.output }}`（AI 輸出含換行符號會破壞 JSON）

### 問題 3：Google Sheets 欄位出現 `=https://...`（多了等號）
- Map Each Column Manually 的值欄位直接填 `{{ $json.url }}`，**不需要** `=` 前綴
- 不要寫 `={{ $json.url }}` 或 `=={{ $json.url }}`（加了等號會輸出 `=https://...`）

### 問題 4：同一留言反覆收到 LINE 通知
- 確認 Code Node 裡的 `allRows` 合併邏輯正確存在（含 `flatMap`）
- 確認 Google Sheets 的 `已回覆留言ID` 欄有確實被寫入（檢查執行紀錄中 `更新已回覆紀錄` 節點）

### 問題 5：有多個規則的同一支影片收到重複通知
- 確認 Code Node 裡有 `sameVideoMinRow` 的 notify 守衛邏輯
- 這確保同一次執行內，只有 row_number 最小的列會發送 notify

### 問題 6：YouTube 回覆失敗（403 Forbidden）
- 確認 Google OAuth2 憑證有加入 `youtube.force-ssl` scope
- 重新授權憑證（Credentials → 重新 Sign in with Google）

### 問題 7：n8n 重啟後 Webhook 失效
- 確認 docker-compose.yml 的 `WEBHOOK_URL` 設定正確
- 確認 ngrok 已重新啟動且 domain 相同（固定 domain 不會變）

---

## 附錄：重要資料彙整

| 項目 | 說明 |
|---|---|
| n8n 網址 | http://localhost:5678 |
| Google Cloud Console | https://console.cloud.google.com |
| Google AI Studio（Gemini） | https://aistudio.google.com |
| LINE Developers | https://developers.line.biz |
| 自己的 YouTube 頻道 ID | 在 YouTube Studio → 設定 → 頻道 → 基本資訊 |
| 自己的 LINE User ID | LINE Developers → Basic settings |

## 附錄：Google Sheets 欄位說明

| 欄位 | 用途 |
|---|---|
| youtube_url | 影片完整網址 |
| ID | 影片 ID（從網址擷取） |
| 關鍵字 | 觸發自動回覆的關鍵字 |
| 回應 | 自動回覆的文字 |
| 檢查時間 | 最後一次處理時間（自動填入） |
| 已回覆留言ID | 已處理過的留言 ID，逗號分隔（自動填入） |
