---
title: "從 1.5GB 到 300MB：我的 Dockerfile 瘦身與多階段構建實錄"
date: 2026-04-30T16:00:00+08:00
draft: false
description: "這份文件詳細說明了如何利用多階段構建 Dockerfile"
tags: ["程式", "部署", "docker"]
series: ["部署"]
---

在後端開發的過程中，你是否也遇過明明程式碼只有幾 MB，但編譯出來的 Docker Image 卻直衝 1.5GB？
這不僅佔用磁碟空間，每次部署到 GCP 或 AWS 時，光是上傳映像檔就等到想喝咖啡。

本文將分享我如何將映像檔體積縮減 80%，並同時兼顧開發環境的便利性。

# 一、 為什麼你的映像檔會爆炸？
在動手優化前，我們先看看到底是誰吃掉了空間。主要原因通常有三個：
1. 基礎映像檔（Base Image）太胖：預設的 python:3.10 基於完整版 Debian，內含許多開發時才需要的編譯工具與資料庫驅動開發庫。
2. 建置工具的殘留：為了安裝套件，我們常在容器內安裝 `gcc`、`uv` 或 `poetry`，但這些工具在程式執行時根本用不到。
3. 測試與雜物包進去了：tests/ 資料夾、`.git` 目錄、甚至是本地端的 `.venv` 被直接 `COPY` 進容器。

# 二、優化方案
更換為 Slim 版基礎映像檔（換成 `python:3.10-slim`）。

- 優點：體積縮減至 100MB — 200MB。
- 風險：有些 Python 套件（如 `pandas`, `cryptography`）需要編譯，在 slim 版會報 `error: gcc not found`。
- **對策**：使用 **多階段建置（Multi-stage Build）**，在 Build 階段編譯，在 Runtime 階段保持乾淨。

## 多階段建置（Multi-stage Build）
為兼顧開發與生產環境，我們將 Dockerfile 切分為「建築工地」與「成屋」：

- Build 階段：安裝 `gcc`、`uv`，下載並編譯所有 Python 套件（`.venv`）。
- Runtime 階段：從工地把編譯好的 `.venv` 搬過來，捨棄掉數百 MB 的編譯工具。

挑戰：如果為了瘦身把 tests/ 和開發工具都刪掉，那我在本地開發時怎麼跑 `pytest`？ 
解法：在同一個 `Dockerfile` 中定義不同的 Target Stages。

### 實戰代碼示範
```ymal
# --- 第一階段：Base / Builder (負責編譯環境與安裝依賴) ---
FROM python:3.10 AS builder
COPY --from=ghcr.io/astral-sh/uv:0.5.11 /uv /uvx /bin/
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

# 安裝所有依賴 (包含 dev 依賴，如 pytest)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project

# --- 第二階段：Development (開發/測試專用) ---
# 這個階段基於胖映像檔，且包含所有程式碼與測試
FROM builder AS development
WORKDIR /app
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH=/app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# --- 第三階段：Production (最終產品瘦身版) ---
# 這個階段換成 slim，只搬運編譯好的產物
FROM python:3.10-slim AS production
WORKDIR /app
# 重新執行一次只裝 production 依賴的 sync (或直接從 builder 複製，但建議分開處理確保乾淨)
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH=/app

COPY ./scripts /app/scripts
COPY ./app /app/app
COPY ./alembic.ini /app/
RUN chmod +x /app/scripts/*.sh

CMD ["uvicorn", "app.main:app", "--workers", "4", "--host", "0.0.0.0", "--port", "8000"]
```

### 如何在 docker-compose 中切換？
透過 `build.target` 參數，我們可以輕鬆切換

本地開發 (`docker-compose.override.yml` 會覆寫 `docker-compose.yml`)：
```ymal
backend:
  build:
    context: ./backend
    target: development # 這裡會包含測試工具，並支援 --reload
  volumes:
    - ./backend:/app     # 熱更新必備
```

正式部署 (`docker-compose.yml`)：
```ymal
backend:
  build:
    context: ./backend
    target: production  # 這裡產出的 Image 只有 300MB
```
使用 `docker compose -f docker-compose.yml up -d — build`（等同先 `build` 再 `up`） 時，測試代碼完全不會被打包進去，實現了完美的隔離。

### .dockerignore
即便 Dockerfile 寫得再好，如果沒有 `.dockerignore`，`COPY . .` 就會把幾百 MB 的本地 `.venv` 或 `.git` 塞進去。

必備的 `.dockerignore` 檔案內容：
```python
.git
.venv
__pycache__
*.pyc
.pytest_cache
tests/   # 在 Production 階段會被排除
```
# 三、結語
透過 Slim 基礎映像檔 + 多階段建置 + Target Stages，我成功將 Image 從 1.46GB 壓縮到了 300MB 左右。
這不僅讓推送到 GCP Artifact Registry 的時間從分鐘縮短到秒，更讓雲端伺服器的部署過程更加順滑。