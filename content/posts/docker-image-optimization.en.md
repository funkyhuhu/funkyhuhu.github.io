---
title: "From 1.5GB to 300MB: A Practical Guide to Dockerfile Slimming and Multi-Stage Builds"
date: 2026-04-30T16:00:00+08:00
draft: false
description: "A detailed walkthrough on optimizing Docker images using multi-stage builds and target stages."
tags: ["Programming", "Deployment", "Docker"]
series: ["Deployment"]
categories: ["Tech"]
---

During backend development, have you ever noticed that even though your code is only a few MBs, the compiled Docker image ends up exceeding 1.5GB?

Not only does this waste disk space, but it also turns every deployment to GCP or AWS into a "coffee break" while waiting for the image to upload. In this post, I’ll share how I reduced my image size by 80% without sacrificing the convenience of my local development environment.

# 1. Why Is Your Image So Bloated?
Before optimizing, let’s identify the culprits taking up space. There are usually three main reasons:

- **Overweight Base Images:** The default python:3.10 image is based on the full Debian distribution. It contains numerous build tools and database driver libraries that are only needed during development.

- **Residual Build Tools:** To install dependencies, we often install `gcc`, `uv`, or `poetry` inside the container. However, these tools are completely unnecessary for running the actual application.

- **Unnecessary Files:** The tests/ folder, `.git` directory, or even the local `.venv` often get accidentally bundled into the container via the `COPY` command.

# 2. Optimization Strategy
The first step is switching to a Slim base image (e.g., python:3.10-slim).

- **Pros:** Reduces base size to roughly 100MB – 200MB.

- **Risks:** Certain Python packages (like `pandas` or `cryptography`) require compilation. Installing them on a slim image will throw an `error: gcc not found`.

**The Fix**: Use **Multi-stage Builds** to compile in a "Build" stage and keep the "Runtime" stage clean.

## Multi-stage Builds
To balance development needs with production efficiency, we split the Dockerfile into a "Construction Site" and a "Finished House":

- **Build Stage:** Install `gcc` and `uv`, then download and compile all Python packages into a `.venv`.

- **Runtime Stage:** Transfer only the compiled `.venv` from the builder and discard hundreds of MBs of build tools.

**The Challenge:** If we delete tests/ and dev tools for the sake of slimming down, how do we run pytest during local development?

**The Solution:** Define different Target Stages within the same Dockerfile.

## Practical Code Example
```YAML
# --- Stage 1: Base / Builder (Handles compilation and dependencies) ---
FROM python:3.10 AS builder
COPY --from=ghcr.io/astral-sh/uv:0.5.11 /uv /uvx /bin/
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

# Install all dependencies (including dev dependencies like pytest)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project

# --- Stage 2: Development (For Dev/Testing) ---
# This stage is based on the builder image and includes all source code and tests
FROM builder AS development
WORKDIR /app
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH=/app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# --- Stage 3: Production (The Slimmed-down Final Product) ---
# Switch to the slim image and only carry over the compiled artifacts
FROM python:3.10-slim AS production
WORKDIR /app
# Re-run sync for production only (or copy from builder, but re-sync ensures a clean environment)
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH=/app

COPY ./scripts /app/scripts
COPY ./app /app/app
COPY ./alembic.ini /app/
RUN chmod +x /app/scripts/*.sh

CMD ["uvicorn", "app.main:app", "--workers", "4", "--host", "0.0.0.0", "--port", "8000"]
```
Switching Environments in `docker-compose`
By using the `build.target` parameter, we can easily toggle between stages.

Local Development (`docker-compose.override.yml` overrides `docker-compose.yml`):

```YAML
backend:
  build:
    context: ./backend
    target: development # Includes test tools and supports --reload
  volumes:
    - ./backend:/app     # Essential for hot-reloading
```
Production Deployment (`docker-compose.yml`):

```YAML
backend:
  build:
    context: ./backend
    target: production  # The resulting image is only ~300MB
```
When running `docker compose up -d --build`, the production image is built without any test code, achieving perfect isolation.

## .dockerignore
No matter how well-written your Dockerfile is, without a `.dockerignore`, `COPY . .` will suck in hundreds of MBs from your local `.venv` or `.git`.

Essential `.dockerignore` content:

```python
.git
.venv
__pycache__
*.pyc
.pytest_cache
tests/   # Excluded in the Production stage
```
# 3. Conclusion
By combining **Slim base images**, **Multi-stage builds**, and **Target Stages**, I successfully compressed my image from 1.46GB down to approximately 300MB.

This optimization reduced GCP Artifact Registry push times from minutes to seconds and made the entire deployment pipeline significantly smoother.