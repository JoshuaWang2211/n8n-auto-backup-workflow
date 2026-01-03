# 💾 Databases Auto Backup – 全自動多資料庫雲端備份（n8n Workflow）

此 workflow 建構於 **n8n**，可每天自動備份你的 n8n workflows、Qdrant 向量資料庫與 Postgres 資料庫，並上傳至 Google Drive 妥善保存。  
**Set and Forget** — 設定完成後無需任何手動介入，系統會自動執行備份與清理舊檔。

---

## 🎯 適用對象

這個 workflow 特別適合：

- **自架派玩家**：喜歡在自己的機器（VPS、NAS、家用伺服器）上架設自動化工具與資料庫的人
- **資料安全控**：擔心本機硬碟故障、伺服器當機導致資料遺失的人
- **懶人自動化愛好者**：希望「設定一次，永遠運作」的極簡派

> 💡 **再也不用擔心電腦或硬碟壞掉** — 所有重要資料每天自動同步到雲端，即使本地硬體完全損毀，也能從 Google Drive 快速還原。

---

## 🔐 Credentials 需求

此 workflow 需要設定以下認證：

| Credential | 用途 | 設定方式 |
|------------|------|----------|
| **Google Drive OAuth2** | 上傳備份檔案、查詢/刪除舊檔案 | n8n Credentials → Google Drive → OAuth2 連線 |
| **Qdrant API Key** | 存取 Qdrant REST API 建立與下載 snapshot | 透過環境變數 `QDRANT__SERVICE__API_KEY` 設定 |

> ⚠️ **匯入注意**：此 workflow 匯入到新的 n8n 實例後，需要重新連結 Google Drive credential，因為 credential ID 是各實例獨立的。

---

## ✨ 功能特色

### 🔄 三合一全自動備份
- **n8n Workflows**：匯出所有 workflow 為 JSON 檔案
- **Qdrant Vector DB**：自動建立所有 collection 的 snapshot
- **Postgres Database**：備份最新的 `.sql.gz` 壓縮檔

### ☁️ Google Drive 雲端同步
- 所有備份檔案自動上傳至指定的 Google Drive 資料夾
- 檔名自動加上日期標記，易於識別與管理
- 檔案格式：`YYYY-MM-DD_workflows.json`、`YYYY-MM-DD-collection_name.snapshot`

### 🧹 自動清理舊備份
- 自動辨識並刪除 **7 天以上** 的舊備份檔案
- 節省雲端儲存空間，無需手動維護
- 刪除的檔案會移至 Google Drive 垃圾桶（可復原）

### ⏰ 每日定時執行
- 每天 **07:50** 自動觸發備份流程
- 四條備份路徑 **並行執行**，效率最佳化
- 若要調整時間可直接編輯 Schedule Trigger 節點

---

## 🧱 Workflow 節點架構總覽

### 1️⃣ 每日排程觸發
**Every Day（Schedule Trigger）**
- 每天早上 07:50 觸發
- 同時啟動四條並行流程

---

### 2️⃣ 流程一：n8n Workflows 備份  
**Export All Workflows → Read JSON File → Upload to GDrive**

- 使用 `n8n export:workflow --all` 指令匯出所有 workflow
- 讀取匯出的 JSON 檔案
- 上傳至 Google Drive 指定資料夾

---

### 3️⃣ 流程二：Qdrant 向量資料庫備份  
**List All Collections → Loop Collections → Create Snapshot → Download Snapshot → Upload to GDrive → Delete Qdrant Temp**

- 透過 Qdrant REST API 取得所有 collection 名稱
- 逐一為每個 collection 建立 snapshot
- 下載 snapshot 檔案
- 上傳至 Google Drive
- 清理 Qdrant 伺服器上的暫存 snapshot

---

### 4️⃣ 流程三：Postgres 資料庫備份  
**Find Latest Backup → Read Backup File → Upload to GDrive**

- 從本機備份目錄找到最新的 `.sql.gz` 備份檔
- 讀取備份檔案為 binary
- 上傳至 Google Drive 指定資料夾

---

### 5️⃣ 流程四：自動清理舊備份  
**Define Folders → Find Files > 7 Days → Delete Old Files**

- 定義三個備份資料夾（Qdrant、Postgres、Workflows）
- 搜尋修改時間超過 7 天的檔案
- 批次刪除舊檔案（移至垃圾桶）

---

## 🚀 使用方式

### 1️⃣ 匯入 workflow JSON
匯入 `[SYSTEM] Databases Auto Backup.json`。

### 2️⃣ 設定 Google Drive 認證  
在以下節點設定你的 Google Drive OAuth2 認證：
- **Upload file**（Qdrant 備份）
- **Upload to GDrive**（Postgres 備份）
- **Upload to GDrive1**（Workflows 備份）
- **Find Files > 7 Days**（查找舊檔案）
- **Delete Old Files**（刪除舊檔案）

### 3️⃣ 設定 Google Drive 資料夾 ID  
修改各 Upload 節點的 `folderId`，指向你的備份資料夾：

```js
// 目前設定
{
  "Qdrant": "1ltO48I28vK8sk8f35MI1AR5fm-YqyPGl",
  "Postgres": "1Gy-v7CdvAzhdT9TFEEdv_V4EmYjfZClr", 
  "Workflows": "13PqpYac0doKW3P_j52a2-g9INpngselh"
}
```

### 4️⃣ 設定 Qdrant API Key  
確保環境變數 `QDRANT__SERVICE__API_KEY` 已正確設定。

### 5️⃣ 確認 Postgres 備份路徑  
確保 Postgres 的定期備份存放於 `/home/node/backups` 目錄。

---

## 📁 備份檔案結構

```
Google Drive/
└── n8n Databases Auto Backup/
    ├── Workflows/
    │   ├── 2025-01-03_workflows.json
    │   ├── 2025-01-02_workflows.json
    │   └── ...
    ├── Qdrant/
    │   ├── 2025-01-03-knowledge_base.snapshot
    │   ├── 2025-01-03-chat_history.snapshot
    │   └── ...
    └── Postgres/
        ├── 2025-01-03-postgres_backup.sql.gz
        ├── 2025-01-02-postgres_backup.sql.gz
        └── ...
```

---

## ⚙️ 進階設定

### 修改備份時間
編輯 **Every Day** 節點：

```json
{
  "triggerAtHour": 7,
  "triggerAtMinute": 50
}
```

### 修改保留天數
編輯 **Find Files > 7 Days** 節點的查詢條件：

```js
// 將 7 改為你想要的天數
modifiedTime < '{{ $now.minus({days: 7}).toISO() }}'
```

---

## 📬 聯絡作者
若你對 n8n、自動化、AI 整合有興趣，歡迎交流：

- GitHub：[@JoshuaWang2211](https://github.com/JoshuaWang2211)

---

本 workflow 旨在提供 **Set and Forget** 的全自動備份體驗，確保你的 n8n 環境資料安全無虞，再也不用擔心意外遺失重要的 workflow 與資料庫內容。
