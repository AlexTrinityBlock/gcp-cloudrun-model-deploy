# 在 GCP CloudRun 上佈署LLM工作負載

## 1. 開啟 CloudShell 並且進行配置

打開 CloudShell 基於網頁的命令行工具，進行以下配置:

```bash
gcloud config set project <your-project-id>
gcloud config set compute/region <your-region>
```

根據文件目前開放區域僅有
us-central1 (愛荷華州)、asia-southeast1 (新加坡)、europe-west1 (比利時)、europe-west4 (荷蘭)

範例:

```bash
# 配置專案，輸入 Project ID
gcloud config set project cloudrun-gpu-lab-20250629
# 配置預設區域，本篇文獻撰寫時，僅開放部分區域
gcloud config set compute/region us-central1
```

## 2. 啟動 Artifact Registry, Cloud Build, Cloud Run, Cloud Storage

CloudRun 是基於容器(Container)的雲端工作負載，所以勢必要有容器儲存庫 Artifact Registry 與容器建構服務 Cloud Build ，還有最終儲存容器的位置 Cloud Storage

所以我們在專案中的 CloudShell 進行啟用上述服務:

```bash
gcloud services enable \
artifactregistry.googleapis.com \
cloudbuild.googleapis.com \
run.googleapis.com \
storage.googleapis.com
```

## 3. 建立 Artifact Registry 容器儲存庫

建立一個 Artifact Registry 容器儲存庫，這是用來儲存我們的容器映像檔的地方，採用以下命令:

```bash
gcloud artifacts repositories create <your-repo-name> \
--repository-format=docker \
--location=<your-region>
```

範例:

```bash
gcloud artifacts repositories create llm-repo \
--repository-format=docker \
--location=us-central1
```

## 4. 建立一個佈署容器用的目錄與 Dockerfile

建立一個目錄來存放我們的 Dockerfile 和相關文件。可以在 CloudShell 中使用以下命令:

```bash
mkdir -p ~/cloudrun-llm
cd ~/cloudrun-llm
```

然後建立一個 Dockerfile，這個檔案將定義我們的容器映像檔。我們採用 Ollama 的容器映像檔來運行 LLM 模型:

```dockerfile
FROM ollama/ollama:latest

# Listen on all interfaces, port 8080
ENV OLLAMA_HOST 0.0.0.0:8080

# Store model weight files in /models
ENV OLLAMA_MODELS /models

# Reduce logging verbosity
ENV OLLAMA_DEBUG false

# Never unload model weights from the GPU
ENV OLLAMA_KEEP_ALIVE -1

# Store the model weights in the container image
ENV MODEL gemma3:4b
RUN ollama serve & sleep 5 && ollama pull $MODEL

# Start Ollama
ENTRYPOINT ["ollama", "serve"]
```

## 5. 建構 CloudRun 容器映像檔

在 CloudShell 中，使用以下命令來建構並佈署容器到 CloudRun:

```bash
gcloud builds submit --tag <region>-docker.pkg.dev/<your-project-id>/<your-repo-name>/llm-container
```

以該專案為例:

```bash
gcloud builds submit --tag us-central1-docker.pkg.dev/cloudrun-gpu-lab-20250629/llm-repo/llm-container --machine-type=e2-highcpu-32
```

## 6. 佈署 CloudRun 容器

在 CloudShell 中，使用以下命令來佈署容器到 CloudRun:

```bash
gcloud run deploy llm-cloudrun \
--image us-central1-docker.pkg.dev/cloudrun-gpu-lab-20250629/llm-repo/llm-container \
--concurrency 4 \
--cpu 8 \
--set-env-vars OLLAMA_NUM_PARALLEL=4 \
--gpu 1 \
--gpu-type nvidia-l4 \
--max-instances 1 \
--memory 32Gi \
--no-allow-unauthenticated \
--no-cpu-throttling \
--no-gpu-zonal-redundancy \
--timeout=600 \
--region us-central1
```

## 7. 透過 CloudRun Proxy 存取 LLM 服務

在 CloudRun 上部署後，可以透過 CloudRun Proxy 存取 LLM 服務。使用以下命令來獲取服務的 URL:

```bash
gcloud run services proxy llm-cloudrun --port=9090 --region=us-central1
```

上述指令會將 CloudRun 服務的流量代理到本地端的 9090 埠，這樣就可以在本地端透過 `http://localhost:9090` 存取 LLM 服務。

開啟另一個 CloudShell 分頁，使用 curl 測試 LLM 服務

串流模式，會一個一個詞回傳

```bash
curl http://localhost:9090/api/generate -d '{
  "model": "gemma3:4b",
  "prompt": "天空還藍嗎?"
}'
```

非串流模式，會一次回傳整個回應:

```bash
curl http://localhost:9090/api/generate -d '{
  "model": "gemma3:4b",
  "prompt": "天空還藍嗎?",
  "stream": false
}'
```

## 小記 - 使用 Gemini CLI 時的 API 金鑰設定

在使用 Gemini CLI 進行操作時，需要設定 API 金鑰。可以在 CloudShell 中設定環境變數。

```bash
export GEMINI_API_KEY=AIzaSyC_6NzAPeFhg2t3oJValSH52HHdkxwMg4A
```