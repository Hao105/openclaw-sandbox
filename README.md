# OpenClaw (小龍蝦) AI Agent 本地隔離部署指南

本專案旨在於高度受限的本機環境中，安全地部署與運行 OpenClaw (小龍蝦) 自主型 AI 代理人。透過 Docker 容器化技術與本機 LLM (Ollama) 的結合，確保所有 AI 運算與資料處理皆在實體與網路雙重隔離的「沙盒」內進行，徹底防範機密外洩與 AI 失控風險。

---

## 🛡️ 核心安全守則 (Security Guidelines)

在操作與擴充本專案時，必須嚴格遵守以下「零信任 (Zero Trust)」原則：

1. **實體與邏輯斷網 (Air-gapped Principle)**
   - 代理人容器所在的 Docker 網路必須永久維持 `internal: true`。
   - 嚴禁掛載任何對外連線的 API Token（如非本機的雲端服務）或提供未經 Proxy 審查的對外連線權限。
2. **最小權限與目錄限縮 (Principle of Least Privilege)**
   - 僅允許 AI 讀寫專屬的 `./workspace` 目錄。
   - 絕對禁止掛載宿主機的根目錄 `/`、`.ssh` 資料夾或任何包含系統環境變數的設定檔。
   - 容器執行必須維持 `privileged: false` 並移除所有非必要的 Linux capabilities (`cap_drop: ALL`)。
3. **資源配額與強制終止 (Resource Limits & Kill Switch)**
   - 必須透過 `deploy.resources` 硬性限制 CPU 與記憶體使用量，防止無窮迴圈耗盡主機資源。
   - 需搭配外部監控腳本，一旦偵測到 CPU 異常飆高或非授權的越權行為日誌，立即觸發 Kill Switch (`docker stop`)。

---

## 💻 軟硬體配置與模型選擇 (Hardware & Model Strategy)

本專案部署針對中高階開發用筆記型電腦 (如配備 Core i7, 48GB RAM, RTX 5050 8GB VRAM 或等值硬體) 進行了最佳化配置：

*   **選用模型：`qwen2.5-coder:7b`**
*   **決策分析 (記憶體溢出效應迴避)：**
    在嘗試部署 32B 級別 (需 ~24GB 記憶體) 的超大模型時，因為 RTX 5050 VRAM 容量限制 (8GB)，系統會自動觸發「VRAM Offloading」機制，將剩餘高達 16GB 的模型參數分流至主機記憶體 (System RAM) 並由 CPU 代為運算。這將導致嚴重的運算延遲，甚至使推論進程觸發 Frontend 的 Timeout 機制而崩潰 (Context Canceled)。
    **解決方案：** 降級採用 7B 版本。7B 模型總佔用約 5GB 左右的記憶體，可 100% 完整載入至獨立顯示卡 VRAM 中，實現極速、零卡頓的 AI 程式碼除錯推論體驗。

---

## 🛠️ 常見問題與進階除錯 (Troubleshooting)

### 重大 Bug 解析：Ollama 127.0.0.1 阻斷連線錯誤

若在啟動後，Docker 日誌頻繁出現 `Ollama could not be reached at http://127.0.0.1:11434.` 的錯誤，這代表前端網頁已失去與背後 Ollama 容器的連結。

*   **根本原因 (Root Cause)：**
    OpenClaw 內部的 LLM 外掛系統對於名稱匹配極為嚴格。若在 `workspace/openclaw.json` 中將本機模型的供應商或 ID 隨意命名 (例如：`ollama_local`)，或者使用了雲端版的標籤 (例如：`qwen2.5/qwen2.5` 會觸發防呆的 API 檢查限制)，系統會因為「找不到精確的外掛名稱」而啟動防護機制，**將連線位址強制退回 (Fallback) 至寫死在程式碼最底層的出廠預設值 `127.0.0.1`**。

*   **解決方案 (The Fix)：**
    1.  **環境變數極限覆蓋**：在 `docker-compose.yml` 中，強制將所有內部可能使用到的路由，皆指向 Docker 內部的 Ollama DNS 名稱。
        ```yaml
        - LLM_PROVIDER=ollama
        - LLM_API_BASE=http://ollama:11434
        - OLLAMA_URL=http://ollama:11434
        - OLLAMA_BASE_URL=http://ollama:11434
        - OLLAMA_HOST=http://ollama:11434
        ```
    2.  **嚴格匹配 JSON 設定**：在 `workspace/openclaw.json` 設定檔中，必須確保 `providers` 的子鍵完全精確為 `"ollama"`，且 `agents.list` 中的模型指派必須使用 `{provider}/{model}` 格式，例如：
        ```json
        "agents": {
          "list": [
            {
              "id": "main",
              "model": "ollama/qwen2.5-coder:7b"
            }
          ]
        }
        ```

---

## 📂 專案結構與資源說明 (Project Structure)

了解專案下的資料夾與核心檔案功能：

*   **`docker-compose.yml`**: 核心部署檔。定義 Postgres、Ollama 與 OpenClaw-agent 的服務配置，包含實體網路隔離、通訊埠號機制。
*   **`workspace/openclaw.json`**: OpenClaw 的中樞微調設定檔。包含介面的來源驗證、模型綁定以及供應商連線參數。
*   **`workspace/`**: 掛載至 OpenClaw 容器內的 `/root/.openclaw`，作為 AI 唯一允許讀寫的「沙盒工作區」，徹底隔離宿主機系統風險。
*   **`ollama_data/`**: 掛載至 Ollama 容器內的存放區。避免 Windows 檔案鎖定造成的 I/O Error。在此永久儲存下載的大型語言模型。
*   **`postgres_data/`**: 對話歷史與系統紀錄持久化資料庫掛載點。

---

## 🚀 部署與啟動指令摘要

1.  **啟動系統**
    ```bash
    docker-compose up -d
    ```
2.  **載入 7B 開發模型**
    ```bash
    docker exec -d local_ollama ollama pull qwen2.5-coder:7b
    ```
3.  **重新載入設定**
    ```bash
    docker restart openclaw_sandbox
    ```
    
至此部署完成。請用瀏覽器存取本機對應的通訊埠進入 Web 介面 (由 docker-compose 定義的 18789 Port)：`http://localhost:18789`