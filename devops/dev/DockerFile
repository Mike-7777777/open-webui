# syntax=docker/dockerfile:1
# Initialize device type args
ARG USE_CUDA=false
ARG USE_OLLAMA=false
ARG USE_CUDA_VER=cu121
ARG USE_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
ARG USE_RERANKING_MODEL=""
ARG USE_TIKTOKEN_ENCODING_NAME="cl100k_base"
ARG BUILD_HASH=dev-build
ARG UID=1000
ARG GID=1000

######## WebUI frontend ########
FROM --platform=$BUILDPLATFORM node:22-alpine3.20 AS build
ARG BUILD_HASH
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ENV APP_BUILD_HASH=${BUILD_HASH}
RUN npm run build

######## WebUI backend ########
FROM python:3.11-slim-bookworm AS base
ARG USE_CUDA
ARG USE_OLLAMA
ARG USE_CUDA_VER
ARG USE_EMBEDDING_MODEL
ARG USE_RERANKING_MODEL
ARG UID
ARG GID

ENV ENV=prod \
    PORT=8080 \
    USE_OLLAMA_DOCKER=false \
    USE_CUDA_DOCKER=false \
    USE_CUDA_DOCKER_VER=cu121 \
    USE_EMBEDDING_MODEL_DOCKER=${USE_EMBEDDING_MODEL} \
    USE_RERANKING_MODEL_DOCKER=${USE_RERANKING_MODEL} \
    # 添加清华镜像源配置
    HF_ENDPOINT=https://hf-mirror.com \
    PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    PIP_TRUSTED_HOST=pypi.tuna.tsinghua.edu.cn

## 其他环境变量保持不变 ##
WORKDIR /app/backend
ENV HOME=/app

# 创建非root用户
RUN addgroup --gid $GID app && \
    adduser --uid $UID --gid $GID --home $HOME --disabled-password --no-create-home app

# 安装系统依赖（精简版）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git build-essential pandoc netcat-openbsd curl jq \
    ffmpeg libsm6 libxext6 && \
    rm -rf /var/lib/apt/lists/*

# 安装Python依赖
COPY --chown=app:app ./backend/requirements.txt .
RUN pip3 install uv && \
    pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu && \
    uv pip install --system -r requirements.txt --no-cache-dir

# 预下载模型（使用清华源）
RUN python -c "import os; \
    from sentence_transformers import SentenceTransformer; \
    SentenceTransformer(os.environ['USE_EMBEDDING_MODEL_DOCKER'], device='cpu')" && \
    python -c "import os; \
    from faster_whisper import WhisperModel; \
    WhisperModel('base', device='cpu', compute_type='int8', \
    download_root=os.path.join('/app/backend/data/cache/whisper/models'))" && \
    python -c "import os; \
    import tiktoken; \
    tiktoken.get_encoding('cl100k_base')"

# 复制前端文件和后端代码
COPY --chown=app:app --from=build /app/build /app/build
COPY --chown=app:app ./backend .

# 设置权限
RUN mkdir -p /app/backend/data && \
    chown -R app:app /app

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl --silent --fail http://localhost:${PORT:-8080}/health | jq -ne 'input.status == true' || exit 1
USER app:app
CMD [ "bash", "start.sh" ]