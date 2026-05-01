---
title: "用 GitHub Actions + Docker 打造可 rollback 的 CI/CD：從 build image 到 production deploy 的實戰思考"
date: 2026-05-01T04:13:00+08:00
draft: false
description: "這份文件詳細說明了如何利用GitHub Actions + Docker 打造可 rollback 的 CI/CD"
tags: ["程式", "部署", "docker", "CI/CD"]
series: ["部署"]
---

在軟體開發中，寫 Code 往往不是最難的，「如何確保寫好的 Code 穩定地跑在 Server 上」才是真正的挑戰。
這次我做的不是「把網站丟上 server」而已，而是把一條完整的部署路徑整理成可以反覆使用的流程：

- PR 進來先驗證
— main branch 自動 build image
— push 到 Docker Hub
— GitHub Actions 透過 SSH 到 VM
— 在 server 上 docker compose pull 與 docker compose up -d

這件事看起來像是把幾個工具串起來，但真正麻煩的從來不是工具本身，而是每一層之間的責任切分有沒有想清楚。
我最後比較在意的不是「有沒有部署成功」，而是這條路徑失敗時，我能不能快速知道到底壞在哪裡。

很多開發者（包括過去的我）在專案初期為了快，習慣用「人肉部署」：
手動 `build Docker image`、手動 `push` 到 Docker Hub、`SSH` 進 Server、重新 `Build` 專案。
這種做法在規模擴大後，會直接導致三個災難：版本不可追蹤、環境不一致、以及幾乎不可能實現的快速 Rollback。

這篇文章分享我如何重新思考 CI/CD 架構，解決部署過程中的核心痛點。

> **部署應該是可重複、可追蹤、可回復的工程流程，而不是人工操作流程**

# 一、 決策：為什麼是 GitHub Actions + Docker？
技術選型不應該是追求流行，而是要權責分明的解決具體問題。

**GitHub Actions**：將 Build 標準化，決定何時 build、何時 deploy
我選擇 GitHub Actions 的原因很簡單：

> 原生整合：它與 Git Repository 深度綁定，當 Code 成為唯一真理，自動化觸發（Event-driven）就是理所當然的結果。
> 可重現性： CI 不應該發生在「我的電腦」，因為我的電腦有特定的環境變數和快取。透過 GitHub Actions，我們強迫 Build 過程發生在一個乾淨、可重現的虛擬環境中。

**Docker**：消除「在我電腦上明明可以跑」，保存可以被拉取的 artifact
在導入 Containerization 之前，最常遇到的問題是 Dependency 版本衝突或 Runtime 不一致。
Docker 對我的核心價值在於：它將執行環境變成了一種可移動的「Artifact（產出物）」。

**Docker Compose**：聲明式部署的平衡點，只負責「把指定版本的 image 跑起來」
為什麼不用 Kubernetes？因為對於中小型 Production，K8s 的維護成本遠超收益。
我選擇 Docker Compose 是為了實現 Declarative（聲明式）部署：
用 YAML 描述狀態，而不是用 Shell Script 描述動作。
部署不再是執行一連串指令，而是告訴 Server：「請確保目前的環境狀態符合這份設定檔。」

這樣切開之後，CI 跟 CD 就不會互相污染。GitHub Actions 不需要知道太多 server 細節，server 也不需要參與 build。

# 二、 核心問題與重新設計：
在優化過程中，我遇到了三個指標性的問題，這些問題本質上都指向同一個架構缺陷。

## 問題 1：`:latest` 標籤導致的不可預測性
早期 Pipeline 習慣直接 `Push image:latest`。這看似方便，卻是 Production 的自殺行為。

- 痛點： `latest` 永遠會被覆蓋，你無法知道現在 Server 上跑的到底是哪個 Commit。
- 思考： 缺乏 Immutable Versioning（不可變版本控制）。如果版本會變，那就無法追蹤，更無法回退。

**解決**：導入 Immutable Image 策略
現在，每一個版本的 Tag 都是唯一的（例如 sha-7ab2f1 或 v1.2.3）。
每個版本都是不可變的 Artifact。這解決了 Rollback 問題 — — Rollback 只是「選回舊版本的 Tag」，不再需要重新 Build 舊 Code。

## 問題 2：CI/CD 職責混亂
我曾嘗試讓 CI 同時處理 Build 與部署邏輯（更新 Server 設定）。

- 痛點： 當 CI 失敗時，我分不清楚是 Build 壞了，還是 Server 連線壞了。
- 思考： 關注點分離（Separation of Concerns）。CI 的任務應止於產出 Image，而不該干涉 Runtime 的複雜狀態。

**解決**：Pull-based Deployment 流程
我將流程簡化為：
CI 階段： 驗證、打包、將帶有 Hash Tag 的 Image 推送到 Docker Hub。
CD 階段： Server 透過環境變數注入（Environment Variables）取得目標 Tag，執行 docker compose pull 並重啟。
核心總結： 部署不再是一個「覆蓋」的過程，而是一個「切換」的過程。

# 四、 最終架構流程圖

{{< mermaid >}}
flowchart TD
    A["GitHub Push / Tag"]
    B["GitHub Actions (CI)"]
    C["Build Docker Image"]
    D["Push to Docker Hub"]
    E["SSH to Server (CD)"]
    F["docker compose pull"]
    G["docker compose up -d"]
    H["Running New Version"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
{{< /mermaid >}}


# 五、 結語

CI/CD 的重點不在於你用了什麼酷炫的工具，而在於你有沒有建立起一套可預測、可重現、可追蹤的發布流程。

當你將部署邏輯從「指令式（手動做什麼）」轉變為「聲明式（我要什麼狀態）」，並透過 Immutable Artifact 確保版本安全時，你才真正擁有了對系統的掌控權。現在，即使發布出現 Bug，我也能在 30 秒內回退到任何一個歷史版本。這份安全感，才是自動化架構真正的價值。

如果你也正在經歷手動部署的痛苦，不妨先從「Image 版本化」這個小點開始改進！