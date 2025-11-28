# 🚀 Nginx 代理 Gemini API (Vertex AI)

本项目提供了一个高性能的 Nginx 反向代理配置，用于解决中国大陆地区访问 Google Vertex AI (Gemini) API 的网络连接问题。

通过将 Nginx 部署到香港（或 GCP 其他境外区域），并将客户端请求转发到 `aiplatform.googleapis.com`，实现稳定、高效的 API 访问。

## 📁 文件结构

```
nginx-proxy-gemini/
├── Dockerfile                  # Docker 镜像构建文件
├── nginx.conf                  # Nginx 主配置文件 (包含性能优化)
├── gemini-proxy.conf           # Nginx 业务配置文件 (核心代理逻辑)
└── README.md                   # 本说明文件
```

## ⚙️ 配置文件详情

### 1\. `nginx.conf` (主配置)

该文件包含高性能的网络优化配置：

  * **工作进程数**：设置为 `auto`，自动匹配 CPU 核心数。
  * **并发连接**：`worker_connections` 设置为 `65535`，支持高并发。
  * **I/O 模型**：使用 Linux 下高效的 `epoll` 模型。
  * **网络优化**：开启 `sendfile on`、`tcp_nopush on` 和 `tcp_nodelay on`，以减少延迟和提高 I/O 性能。
  * **请求体大小**：`client_max_body_size` 设置为 `1024m`，以支持大型模型请求。
  * **日志级别**：`error_log` 设置为 `warn`，以降低日志 I/O 压力。

### 2\. `gemini-proxy.conf` (代理配置)

该文件是核心代理逻辑，专门用于转发 Gemini API (Vertex AI) 的请求。

  * **监听端口**：`listen 8080`。这是 Cloud Run 默认监听的端口。
  * **代理路径**：`location /v1` 用于匹配 Gemini API 的路径。
  * **目标地址**：`proxy_pass https://aiplatform.googleapis.com/v1`。
  * **流式优化**：
      * `proxy_buffering off;` 关闭缓冲，实现流式输出（如 `stream=True`）的低延迟。
      * `proxy_read_timeout 600s;` 延长读取超时时间，防止生成长内容时连接断开。

## 📦 Docker 镜像构建

### `Dockerfile`

```dockerfile
FROM nginx:alpine

# 删除默认配置
RUN rm /etc/nginx/conf.d/default.conf
RUN rm /etc/nginx/nginx.conf 

# 复制自定义配置文件，覆盖默认设置
COPY gemini-proxy.conf /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/nginx.conf
```

## ☁️ Google Cloud 部署指南

### 预备工作

请在开始前安装 Google Cloud CLI，并替换以下环境变量：

  * `YOUR_PROJECT_ID`：您的 GCP 项目 ID
  * `YOUR_LOCATION_OF_REPO`：您的 Artifact Registry 仓库位置（例如：`asia-east1`）
  * `REPOSITORY_NAME`：您为 Docker 仓库定义的名称

### 1\. 创建 Artifact Registry 仓库

```bash
# 启用 Artifact Registry 服务
gcloud services enable artifactregistry.googleapis.com

# 创建 Docker 仓库
PROJECT_ID=YOUR_PROJECT_ID
LOCATION=YOUR_LOCATION_OF_REPO
REPOSITORY_NAME=REPOSITORY_NAME

gcloud artifacts repositories create $REPOSITORY_NAME --repository-format=docker \
    --location=$LOCATION --description="Docker repository" \
    --project=$PROJECT_ID
```

### 2\. 构建镜像并推送

首先克隆本项目，然后使用 Cloud Build 构建并推送镜像。

```bash
# 克隆源代码
git clone https://github.com/PrismaCloudAI/nginx-proxy-gemini.git
cd nginx-proxy-gemini

# 构建镜像并推送到 Artifact Registry
gcloud builds submit --region=$LOCATION --tag $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/nginx-proxy-gemini:v1 .
```

### 3\. 部署到 Cloud Run

将构建好的镜像部署到 Cloud Run 服务。

```bash
# 部署容器
# Cloud Run 服务名为 vertex-ai-proxy 
# --port 80 在此案例中应改为 --port 8080 以匹配 nginx 配置 (gemini-proxy.conf 中监听的端口)
gcloud run deploy vertex-ai-proxy \
    --region $LOCATION \
    --port 8080 \
    --image $LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY_NAME/nginx-proxy-gemini:v1
```

## 💻 代码调用示例

部署成功后，您可以通过 Python SDK 或任何 HTTP 客户端访问 Cloud Run 服务的端点，实现对 Gemini API 的间接访问。

**注意：** 您必须在 `genai.Client` 的 `http_options` 中指定 Cloud Run 服务的 URL。

```python
from google import genai
from google.genai import types
from datetime import datetime

# 替换为您的 Cloud Run 服务的公开 URL（例如：https://vertex-ai-proxy-xxxxxx-xx.run.app）
endpoint = 'https://nginx-gemini-proxy-765863079108.asia-east2.run.app'

client = genai.Client(vertexai=True, http_options={
        "base_url": endpoint,
        # 延长超时时间以适应代理和长文本生成
        "timeout": 600000, 
    },    
    # 如果用服务账号认证（ADC方式），请填入以下参数
    project=YOUR_PROJECT_ID,
    location="global" 
    # 如果是api_key认证，请使用下面的参数并去掉project和location参数
    # api_key=YOUR_API_KEY
)

# 示例：使用 Gemini Pro 模型
response = client.models.generate_content(
    model='gemini-2.5-pro',
    contents='写一首关于技术进步的五言绝句。'
)

print(response.text)
```
