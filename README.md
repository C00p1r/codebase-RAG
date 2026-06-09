# codebase-RAG

---

## 🛠️ 開發環境心法：Colab 與 GitHub 的角色分工

* **GitHub（原始碼託管）**：存放你的 RAG 系統主程式（Python 腳本或 Notebook），同時也是你用來**測試增量更新（Git Diff）的目標 Codebase 來源**。
* **Google Colab（運算與實驗）**：負責執行 Python 腳本。因為 Colab 每次重開機會清空環境，我們必須透過 Git 指令動態抓取程式碼，並將持久化的資料（如 Vector DB）儲存在 **Google Drive** 或記憶體中。

---

## 📋 Google Colab + GitHub 實作工作流程

### 🚀 階段一：環境初始化與資料庫持久化

在 Colab 筆記本的開頭，你必須先完成 Git 授權與掛載雲端硬碟，確保資料庫不會因為連線中斷而消失。

1. **掛載 Google Drive**：用來存放持久化的向量資料庫（如 ChromaDB/FAISS 的本底檔案），避免 Colab 檔案過期被清空。
2. **複製（Clone）你的專案倉庫**：
* 將你寫好的 RAG Pipeline 主程式從 GitHub Clone 到 Colab 空間。
* 同時下載你要拿來當測試對象的小型開源 Codebase（例如：一個小型 C/Python 專案）。



---

### 🔍 階段二：底層解析與動態切塊 (步驟 1 & 2)

* **任務 1：安裝環境與載入 Grammar**
* 在 Colab 中用 `!pip install tree-sitter tree-sitter-python tree-sitter-c` 安裝套件。
* 寫一段初始化腳本載入目標語言的 Parser。


* **任務 2：走訪 AST 與 Function 等級切塊**
* 利用遞迴或佇列（Queue）遍歷語法樹節點，判斷 `node.type == 'function_definition'` 。


* 透過 `node.start_byte` 與 `node.end_byte` 擷取完整的程式碼字串，存入 Python 字典（Dictionary）中準備處理 。





---

### 🔄 階段三：Git 差異比對與資料庫 Upsert (步驟 4 提前實作)

這是你專題最硬核的「系統工程」核心，建議在設計資料庫時就直接加入此邏輯。

* **任務 1：Hash 算力與 Metadata 設計**
* 針對切下來的每一個 Function 區塊字串，呼叫 Python 內建的 `hashlib.md5(text.encode('utf-8')).hexdigest()` 算出雜湊值 。


* 定義唯一的 Unique ID（格式：`"{file_path}::{function_name}"`） 。




* **任務 2：利用 Git 追蹤變更**
* 當你的測試 Codebase 有變更時，在 Colab 執行 `!git diff --name-only HEAD~1`（或與前一次提交比對），找出被修改的檔案清單。


* **任務 3：智慧型 Upsert**
* 
**新增/修改**：Tree-sitter 重新解析該檔案，若算出新的 MD5 Hash 與 Vector DB 內的不符，則呼叫 `vector_db.upsert()` 覆蓋舊向量與原始碼 。


* **刪除**：若 Git 顯示檔案刪除，直接透過 Metadata 篩選 `file_path` 批次刪除該檔案的所有 Chunks。



---

### 🧠 階段四：向量化與知識庫建立 (步驟 3 & 4)

* **任務 1：串接 Embedding 模型**
* 
**本地免錢方案**：用 `!pip install sentence-transformers`，下載 `BAAI/bge-large-en-v1.5`，直接用 Colab 的免費 CPU/GPU 進行推論（幾毫秒即可完成） 。


* 
**雲端 API 方案**：安裝 `openai` 套件，設定好 `OPENAI_API_KEY`，呼叫 `text-embedding-3-small` 。




* **任務 2：寫入持久化資料庫**
* 使用 ChromaDB 或 FAISS，將資料庫的 `persist_directory` 設定在第一階段掛載的 **Google Drive 資料夾**內，確保向量資料永久保存 。





---

### 💬 階段五：在線檢索與 LLM 拼裝生成 (步驟 5 & 6)

* **任務 1：問題向量化與相似度檢索（Top-K）**
* 將使用者的輸入問題（如 `"schedule() 如何挑選 task？"`）丟進同一個 Embedding 模型轉為向量 。


* 調用資料庫的查詢方法（如 `db.similarity_search_by_vector`），設定 `k=3`，精準撈出最相關的 3 個 Function 片段 。




* **任務 2：動態範本拼裝（Prompt Engineering）**
* 在 Python 中寫死一個 System Prompt 範本，利用字串格式化（`f-string`）將撈出來的程式碼與問題無縫塞入 。




* **任務 3：呼叫 LLM API 輸出**
* 將拼裝好的超級 Prompt 透過 API 送給 ChatGPT 或 Claude，並在 Colab 畫面上印出最終答案 。





---

## 📅 單人實作時程建議表 (搭配 Colab 快速開發)

| 週次 | 核心任務 | Colab / GitHub 實作重點 |
| --- | --- | --- |
| **W1~W2** | **環境建立與 AST 跑通** | 在 Colab 成功用 Tree-sitter 切出第一個 Function，並 commit 腳本到 GitHub 。

 |
| **W3~W4** | **資料庫與向量串接** | 成功將 Chunks 轉成向量，存入 Google Drive 的 ChromaDB，並實作 Hash 機制 。

 |
| **W5** | **Git 增量更新機制** | 寫出一段 Python 腳本，能自動讀取 `git diff` 並完成資料庫的 `upsert`，這是你的核心創新點 。

 |
| **W6** | **檢索與 LLM 串連** | 跑通完整的「問題 -> 檢索 -> 拼裝 -> LLM 回答」Pipeline 。

 |
| **W7+** | **優化與 UI 展示** | 建立 30 題測試集測試 Recall ，並用 `ngrok` 串接 Colab 把 Streamlit 介面架起來對外展示 。

 |

這個流程繞開了你自己寫神經網路的巨大坑洞，讓你聚焦在「資料如何精準流轉、如何節省計算成本（增量更新）」的**系統工程師**思維 。用 Python 來寫這套邏輯，在 Colab 的環境下是非常直觀且易於偵錯的！
