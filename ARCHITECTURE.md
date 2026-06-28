# FALO AI Node - 系統架構說明書 (ARCHITECTURE.md)

本文件定義了 **FALO AI Node** 的專案架構、設計原則與技術路線藍圖。此架構專為快速驗證的 PoC/MVP 階段設計，同時具備高度擴充彈性，以利未來無縫遷移至雲端無伺服器 (Serverless) 架構。

---

## 📌 技術路線藍圖 (Technical Roadmap)

目前專案劃分為兩個發展階段，第一階段著重於快速推出產品驗證 (PoC / MVP)，第二階段則針對高併發、雲端代管進行優化。

```mermaid
graph TD
    subgraph Builder_Layer [核心生成層 - 保持不變]
        AI_Builder[AI Builder]
        HTML_Builder[HTML Builder]
    end

    subgraph Interface_Layer [介面抽象層 - 預留擴充]
        Pub_Interface["Publisher Interface (發布器介面)"]
    end

    subgraph Phase_1 [第一階段: PoC / MVP (目前實作)]
        direction TB
        Python_Pub["Windows Python Publisher"]
        Python_Host["Local HTML Hosting (HTTP Server)"]
        Python_API["Python Router / API"]
        QRCode_Gen["Local QRCode Service"]
    end

    subgraph Phase_2 [第二階段: 雲端遷移 (未來架構)]
        direction TB
        CF_Workers["Cloudflare Workers Publisher"]
        CF_Pages["Cloudflare Pages Hosting"]
        CF_KV["Cloudflare KV Router / API"]
        CF_QR["Cloudflare QR Service"]
    end

    Builder_Layer --> Pub_Interface
    Pub_Interface -.->|目前實作| Phase_1
    Pub_Interface -.->|未來升級| Phase_2
```

### 1. 第一階段：PoC / MVP (目前採用)
* **執行環境**: Windows 本地伺服器（FALO AI Node）
* **核心技術**: Python Web Server (例如 FastAPI, Flask, 或原生的 HTTP Server)
* **實作功能**: 
  * **AI Builder**: 處理 AI 邏輯與生成內容。
  * **HTML Builder**: 將生成的素材轉化為 HTML 網頁。
  * **Image Publisher & QRCode**: 本地圖片發布與 QR Code 條碼生成服務。
  * **Local HTML Hosting**: 本地端的 HTML 網頁代管與路由服務。
  * **管理 API**: 提供 HTTP API 供管理頁面調用。

### 2. 未來升級階段：Serverless 遷移
* **升級目標**: 替換為 **Cloudflare Workers** (邊緣運算) + **Cloudflare Pages** (網頁代管)。
* **遷移範圍**: 替換 `Publisher`、`Route` 與 `Hosting` 的具體實作，而**完全不修改**核心生成器 (`Builder` 邏輯)。

---

## 🎨 設計原則 (Design Principles)

為確保「Windows Python」未來能無縫替換成「Cloudflare Workers」，設計時嚴格遵循以下原則：

### 1. 組件化 (Component-based)
所有的模組（如：生成網頁、產出 QR Code、發布圖片、API 路由）必須設計為獨立的組件，禁止交叉依賴或寫死路徑。

### 2. 低耦合 (Low Coupling)
* **解除 Python 系統強綁定**: 不要將發布機制（Publisher）、靜態代管（Hosting）與路由（Route）直接與 Python 的系統底層（如 Python 的特定檔案寫入方法、特定套件或執行緒）寫死。
* **資料驅動**: 所有組件之間僅透過 JSON 格式的數據 payload 進行通訊。

### 3. 介面優先 (Interface-first)
核心的 `Builder` 不直接呼叫具體的儲存或發布服務，而是透過一個抽象的 `Publisher Interface`。

---

## 抽象介面設計與通訊協定 (Interface & API Design)

不論後端是 Python 還是 Cloudflare Workers，發布器均需符合以下介面定義：

### 1. 虛擬介面定義 (Python 範例)
```python
class BasePublisher:
    """
    發布器抽象介面 (未來 Cloudflare Workers 亦需符合此輸入輸出協定)
    """
    def publish_html(self, project_id: str, html_content: str) -> dict:
        """
        發布 HTML 網頁並返回存取網址
        """
        raise NotImplementedError

    def publish_image(self, project_id: str, image_bytes: bytes) -> dict:
        """
        發布圖片並返回存取網址
        """
        raise NotImplementedError

    def generate_qrcode(self, target_url: str) -> str:
        """
        產生指定 URL 的 QR Code 圖片連結
        """
        raise NotImplementedError
```

### 2. 標準 API 規格 (Payload Format)
Builder 發送至 Publisher 的請求格式應標準化為 JSON：

* **發布 HTML 請求**:
  ```json
  {
    "action": "publish_html",
    "project_id": "line-overload-assistant",
    "html_content": "<!DOCTYPE html><html>...</html>",
    "metadata": {
      "title": "LINE 教材首頁",
      "updated_at": "2026-06-28T13:54:00Z"
    }
  }
  ```
* **發布成功回應**:
  ```json
  {
    "status": "success",
    "project_id": "line-overload-assistant",
    "published_url": "http://localhost:8000/shares/line-overload-assistant/index.html",
    "qrcode_url": "http://localhost:8000/qrcode/line-overload-assistant.png"
  }
  ```

---

## ⚠️ MVP 開發規範說明 (MVP Guidelines)

1. **拒絕過度設計 (No Premature Optimization)**
   * **不要**在此階段開始編寫任何 Cloudflare Workers 程式碼，也不要配置 Cloudflare 帳號或環境。
   * **所有功能**在 Windows Python 伺服器本地端可以正常執行、產出 QR Code 並提供瀏覽即可。
2. **保持 Interface 乾淨**
   * 在編寫 Python Server 時，將發布邏輯獨立至 `publishers/` 資料夾，讓該模組只做「接收檔案、存檔、回傳網址」的單純操作，以利第二階段直接重寫該模組為 CF Workers 呼叫。
