---
title: "為什麼我捨棄 Nginx 選擇 Traefik？雲端部署的自動化與開發環境解構"
date: 2026-05-01T04:00:00+08:00
draft: false
description: "這份文件詳細說明了 Nginx 與 Traefik 的差異以及雲端部署的自動化與開發環境"
tags: ["程式", "Nginx", "Traefik"]
series: ["程式"]
---

在處理 GCP VM 部署時，最直覺的作法通常是裝個 Nginx 做 Reverse Proxy，再手動申請 Let’s Encrypt。
但當專案需要頻繁更迭、且必須兼顧「本地開發體驗」與「雲端自動化」時，傳統的做法顯得過於笨重。

這篇文章分享我在專案中，如何透過 GCP + Docker Compose + Traefik 建立一套自給自足的基礎設施，
以及我在處理「環境一致性」與「網路路由」時的思考過程。

# 一、 技術選型：為什麼是 Traefik 而不是 Nginx？
在我的架構設計中，流量路徑是：`Internet -> Traefik -> web (App)`。

很多人會問：既然 web container 裡已經有 Nginx 處理靜態檔案，為什麼外面不直接再套一層 Nginx？

## 1. 動態配置的思考
傳統 Nginx 作為 Reverse Proxy 時，每當新增一個服務或更換 Domain，我必須手動修改 `.conf` 並 `reload`。
而 Traefik 是專為 Docker 生態系設計的，它能透過監聽 Docker Label 自動發現服務。

## 2. HTTPS 證書的自動化管理
Traefik 內建支援 ACME 協議（Let’s Encrypt）。
在 VM 上，我只需要定義好 ACME_EMAIL，它就會自動處理證書申請與過期更新。這讓我能把精力花在功能開發，而不是維護伺服器的憑證。

# 二、 部署挑戰：如何優雅地解決「開發」與「生產」的環境差異？
在本地開發（Local）與雲端部署（VM）之間，最大的痛點在於： 
我想要在本機測試 Traefik 的路由邏輯，但我不希望在本機搞 HTTPS 證書。

**遇到的問題：** 如果只寫一份 docker-compose.yml，我必須不斷修改標籤（Labels）來切換開發環境的網域與 Production 的 HTTPS 設定。這極度容易造成誤刪或設定錯誤。

**解決方案：** 利用 Override 機制實現「配置繼承」。我選擇利用 Docker Compose 預設的 Override (疊加) 特性：
> - `docker-compose.yml (Base)`： 定義 Production 環境。走真實 Domain、開啟 HTTPS。
>
> - `docker-compose.override.yml (Local)`：僅限本機執行。

- 覆蓋 Production 設定：關掉 HTTPS，改走純 HTTP。
- 路由改向：使用 `localhost.tiangolo.com`（一個現成指向 `127.0.0.1` 的網域），確保開發時也有完整的域名測試環境。
- 開發加速：直接將本地的 `index.html` 掛載（`mount`）進 `Container`，實現即時修改即時生效，無需反覆 `Build Image`。

**決策邏輯：** 在 VM 上，我強制作派 `-f docker-compose.yml` 來忽略 `override` 檔；在本機則直接 `docker compose up`。這種「白名單」式的管理確保了線上環境的穩定。

# 三、 GCP VM 部署的實戰細節
在 GCP VM 部署時，我並非手動登入去複製檔案，而是透過環境變數管理部署參數。

## 更新邏輯
對於 Image 的更新，我採用 `pull` -> `up -d` 的策略。
這能確保 Docker 優先從 Registry 抓取最新標籤，並在 Container 層級進行無痛切換，最小化服務中斷時間。

# 四、 結語：思考重於工具
技術選型不應該只是因為「某個工具很紅」，而是要看它解決了什麼問題。

選擇 Traefik 是為了降低維護成本；選擇 Compose Override 是為了極大化開發效率。透過這套架構，我實現了：

- 基礎設施即代碼 (IaC)：所有的路由邏輯都寫在 Compose Labels 裡。
- 開發環境高度一致：本機跑的路由邏輯跟雲端幾乎相同，只是差在有無加密。
- 這是我在專案初期花最多時間思考的地方：如何建立一個讓「未來的我」感到輕鬆的部署環境？