Agentic Reviewer：混合雲架構部署計劃書 (V1.0)
1. 執行摘要 (Executive Summary)
本計劃旨在建構一個尖端的醫療器材法規評論平台。系統採用混合雲架構 (Hybrid Cloud Architecture)：
本地端 (On-premise): 利用 NVIDIA DGX1 的雙 V100 GPU 算力，部署本地大語言模型 (Llama 3 / Gemma 2)，負責處理高隱私、高運算量的醫療器材法規文本分析。
雲端 (AWS): 部署基於 Streamlit 的「Sleek Interface」前端應用，提供極致的用戶體驗與全球存取能力。
中介層 (Connectivity): 透過安全隧道 (Cloudflare Tunnel) 與 模型上下文協議 (MCP/API) 實現本地與雲端的無縫對接，並實施嚴格的身份驗證控制。
2. 本地伺服器環境配置 (NVIDIA DGX1)
DGX1 雖然是前幾代的產品，但雙 Tesla V100 (32GB VRAM each) 依然是極強的推理引擎。
2.1 硬體與操作系統準備
OS: 建議安裝 Ubuntu 22.04 LTS 或最新的 DGX OS。
驅動程式: 安裝 NVIDIA Driver 535 或更高版本，確保支援 CUDA 12.x。
容器化: 安裝 Docker 與 NVIDIA Container Toolkit，這是部署模型最穩定的方式。
2.2 本地 LLM 推理引擎：vLLM 部署
為了最大化 V100 的並行處理能力，我們選用 vLLM 作為推理後端，它支援分散式推理並提供 OpenAI 兼容 API。
模型選擇: Llama-3-70B (需量化) 或 Gemma-2-27B。考慮到 V100 的 VRAM (共 64GB)，建議使用 AWQ 或 GGUF 量化模型以節省空間並提高吞吐量。
部署指令示例:
code
Bash
docker run --gpus all -p 8000:8000 \
vllm/vllm-openai:latest \
--model casperhansen/llama-3-70b-instruct-awq \
--tensor-parallel-size 2 \
--max-model-len 8192
註：--tensor-parallel-size 2 將模型分佈在兩張 V100 上。
2.3 建立 MCP / API 轉發層
為了符合「Agentic Control」需求，我們在 vLLM 前端建立一個輕量級的 FastAPI 封裝，作為 Model Context Protocol (MCP) 伺服器，處理：
請求過濾: 確保法規文本格式正確。
終端控制 (Stop Mechanism): 監控請求狀態，當 AWS 用戶點擊 "Stop" 時，立即終止本地推理進程以釋放 GPU。
3. 雲端佈署與網路安全 (AWS Integration)
3.1 安全通道：Cloudflare Tunnel
由於本地伺服器通常位於防火牆後，直接開放 Port 是危險的。我們使用 Cloudflare Tunnel (Argo Tunnel)。
本地端: 安裝 cloudflared。
設定: 將本地 localhost:8000 映射至一個加密域名（如 api.your-medical-agent.com）。
優勢: 無需開放公網 IP，內建 DDoS 防護，且支援 JWT 驗證。
3.2 權限控制 (Authority Control)
為了防止未授權存取本地 GPU 資源：
API Key 驗證: 在 FastAPI 層實施 X-API-KEY 檢查。
AWS 側: 在 Streamlit App 中配置環境變數，所有對本地 API 的請求必須夾帶金鑰。
OIDC 整合: 若需更高級權限，可於 AWS 部署 Amazon Cognito，僅允許通過公司認證的用戶存取 Web App。
3.3 AWS Streamlit 部署
服務選擇: AWS App Runner (推薦，自動擴展且易於管理) 或 EC2 (t3.medium)。
容器化部署: 撰寫 Dockerfile 封裝 Streamlit 應用。
SSL: 透過 AWS Certificate Manager (ACM) 確保全站 HTTPS。
4. Web App 技術架構：Sleek Interface
本節詳述如何將您要求的「櫻花主題 (Sakura)」與「Bento Grid」落實於 Streamlit。
4.1 UI 實作：自定義 CSS 注入
Streamlit 預設佈局較為單一，需透過 st.markdown(unsafe_allow_html=True) 注入 CSS。
code
Python
# 櫻花主題 (Sakura Palette) 定義
sakura_css = """
<style>
    :root {
        --primary: #f8bbd0;
        --accent: #ff80ab;
        --surface: #ffffff;
    }
    .stApp {
        background-color: var(--surface);
    }
    /* Bento Grid 模擬 */
    .bento-container {
        display: grid;
        grid-template-columns: 3fr 6fr 3fr;
        gap: 15px;
    }
    .glass-card {
        background: rgba(255, 255, 255, 0.7);
        backdrop-filter: blur(12px);
        border-radius: 15px;
        padding: 20px;
        border: 1px solid rgba(248, 187, 208, 0.3);
    }
</style>
"""
4.2 核心組件開發
左側 Studio (3/12): 使用 st.sidebar 或自定義 st.expander 放置模型參數（Temperature, System Prompts）。
中間 Stage (6/12): 放置主文本框與 Start/Stop 雙按鈕。
右側 Monitor (3/12): 實作 Live Log。使用 st.empty() 動態更新 Agent 的思考進程。
4.3 代理人邏輯與 "Stop" 機制
在 Python 後端，我們利用 requests 的 stream=True 功能結合 Python 的執行緒控制：
中斷邏輯: 當用戶點擊 "Stop"，前端發送信號至後端 Thread，調用本地 API 的 Abort 信號。
多代理協作:
Summary Agent: 調用本地 Llama-3 進行長文本摘要。
Visualization Agent: 調用 Gemini 2.5 Flash (雲端 API) 產出 JSON 格式的圖表數據。
Skill Agent: 結合兩者結果，生成最終的 .md 協議。
5. WOW 視覺化實作 (Visualization Engineering)
為了達成企業級的「震撼感」，我們採用以下技術：
動態指標: 使用 streamlit-echarts 或 plotly 實作脈動 (Pulse) 效果的儀表板。
慶祝效果: 整合 streamlit-confetti。當 runFullWorkflow 成功回傳 200 狀態碼時，觸發粉色櫻花色系的紙屑噴灑。
Bento 圖表: 在結果頁面，使用 2x2 的網格展示：
Radar Chart: 顯示合規性覆蓋範圍。
Bar Chart: 顯示風險等級分佈。
Execution Insights: 顯示 AI 信心評分。
6. 混合雲工作流程圖
用戶端: 在 AWS Streamlit App 輸入醫療器材說明書。
應用層: App 將大型法規比對任務發送至 DGX1 (本地)，將 UI 渲染與圖表任務發送至 Gemini 2.5 (雲端)。
安全層: 所有跨雲請求通過 Cloudflare Tunnel 加密，並通過 API Key 驗證身份。
渲染層: 結果回傳 Streamlit，以櫻花主題的 Bento Grid 呈現給用戶。
7. 維運與監控 (Monitoring)
GPU 監控: 在本地安裝 prometheus-node-exporter 與 nvidia-smi 監控看板，防止 V100 過熱。
日誌管理: AWS CloudWatch 記錄所有 Web 端錯誤，本地 vLLM 日誌則紀錄模型推理延遲。
成本控制: 由於主要算力在本地 DGX1，AWS 僅產生極低的運算費用，Gemini 2.5 Flash 的 Token 成本也遠低於 Pro 版本。
8. 結論 (Conclusion)
本方案巧妙結合了本地 DGX1 的算力優勢與 AWS 的靈活性。透過「Sleek Interface」的設計語言，我們不只提供了一個分析工具，更創造了一個專業、優雅且具備高度透明度（透過 Live Log 與 Stop 機制）的法規作業環境。這將顯著提升醫療器材法規從業人員的工作效率，同時確保敏感數據留存在本地端。
9. 20 個深度隨訪問題 (Follow-up Questions)
硬體效能: 若本地 Llama-3-70B 推理速度低於 10 tokens/sec，我們該如何調整 vLLM 的量化參數 (Quantization)？
連線穩定性: 當 Cloudflare Tunnel 發生網路抖動時，Streamlit 前端如何實作斷線重連與狀態保留？
安全性: 除了 API Key，是否需要實施 IP 白名單（將 AWS 出口 IP 加入本地伺服器白名單）以增強防禦？
模型優化: 對於醫療器材法規，是否需要對本地 Llama 模型進行 LoRA 微調，以增加其對 FDA/MDR 術語的理解？
Stop 機制: 如果 vLLM 不直接支援 Abort API，我們該如何在系統層級強制結束 GPU 進程而不影響其他併發請求？
UI 體驗: 在 Streamlit 中，如何確保 Bento Grid 在不同尺寸的顯示器（特別是 13 吋筆電與 27 吋顯示器）上保持比例不跑偏？
併發處理: 當多個用戶同時點擊 "Start Agent" 時，DGX1 的顯存管理策略 (KV Cache) 應如何設定以避免 OOM？
數據隱私: 是否有法規要求特定類型的文件（如臨床試驗數據）絕對禁止傳輸至雲端 Gemini API？
視覺化細節: 在 "WOW Page" 中，如何將 Markdown 報告與動態 Recharts 圖表優雅地結合成一個可下載的 PDF？
Gemini 備援: 若 Google API 發生全球性故障，系統是否具備自動切換機制，讓本地模型接管圖表 JSON 的生成？
版本控制: 系統指令 (System Instructions) 的版本更新如何與 Web App 的部署解耦？是否需要一個本地資料庫存儲 Prompt 歷史？
緩存策略: 對於重複性高的法規條文，是否應在本地實施 Redis 緩存以降低 GPU 負載？
翻譯品質: 繁體中文 (zh-TW) 的翻譯精度在本地 Llama 與雲端 Gemini 之間有何差異？如何保持術語一致性？
日誌分析: Live Log 是否應該存儲到資料庫中，供後續分析 AI 代理人在哪一個步驟最容易出錯？
主題切換: "Jackslot" 主題切換組件在 Streamlit 中如何透過 Session State 實時改變全域 CSS 變數？
負載平衡: 未來若增加第二台 DGX 伺服器，API 轉發層應如何實作 Load Balancing？
異步作業: 對於超過 10,000 字的長文件，Streamlit 應如何處理後端長時間運算而不導致瀏覽器連線逾時？
動畫效能: Framer Motion (motion/react) 在 Python Streamlit 框架下的兼容性如何？是否需要封裝為自定義 Component？
用戶反饋: 系統是否需要一個 "Like/Dislike" 機制來收集用戶對 Agent 評論品質的評分？
未來演進: 隨著 OpenAI 模型上下文協議 (MCP) 的普及，本架構如何快速適應新的標準化接口？
