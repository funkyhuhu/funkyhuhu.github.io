---
title: "精簡 Docker Compose YAML：使用 Anchors (錨點) 與 Aliases (別名)"
date: 2026-04-30T16:00:00+08:00
draft: false
description: "這份文件詳細說明了如何利用 YAML 的 Anchors (&) 與 Aliases (*) 特性來優化 Docker Compose 設定檔"
tags: ["程式", "部署", "docker"]
series: ["部署"]
categories: ["技術"]
---

在維護複雜的 Docker Compose 專案時，我們常會遇到「多個服務使用相同映像檔、網路或環境變數」的情況。
如果每個服務都手寫一遍配置，不僅檔案變得臃腫，未來修改時也容易漏掉其中一項。

為了優化這種重複勞動，最好的精簡方式是使用 YAML 的原生特性：Anchors (錨點 &) 與 Aliases (別名 *)。

# 一、 現狀痛點：重複冗長的配置
當我們有多個服務（如 Web 與 Worker）共用相同基礎設定時，常見的 YAML 寫法如下：
```ymal
services:
    web:
        image: my-app
        environment:
            DEBUG: "true"
            DB_HOST: "postgres"

    worker:
        image: my-app
        environment:
            DEBUG: "true"
            DB_HOST: "postgres"
```
這樣寫的缺點很明顯：一旦 DB_HOST 需要更改，你必須同步手動修改兩個地方。

# 二、 解決方案：定義模板與引用
透過 YAML 的錨點功能，我們可以定義一個核心模板，讓其他服務直接繼承並視需求覆寫。

## 優化後的寫法：
```ymal
# 1. 定義模板 (以 x- 開頭是 Docker Compose 慣例，代表不會被當作服務解析的自定義擴充)
x-common-config: &backend-common
    image: my-app
    environment:
        DEBUG: "true"
        DB_HOST: "postgres"

services:
    web:
        <<: *backend-common # 2. 引用並展開模板內容
        environment:
            <<: *backend-common # 展開環境變數
            PORT: 8000          # 額外增加 web 特有的變數

    worker:
        <<: *backend-common # 3. 直接引用，無須額外設定
```

# 三、 本次優化的三大關鍵

## 1. 結構化管理 (Anchors & Aliases)
透過 `&` 定義錨點名稱，再透過 `*` 進行引用。使用 `<<:` 符號則代表「合併」，
它能將模板內容展開到當前區塊，並允許你在下方繼續添加該服務專屬的配置。
這通常能讓大型專案的檔案長度縮減 40% 以上。

## 2. 網路與權限最小化
在優化邏輯時，我們不應將所有服務都暴露在同一個網絡下。
- 內部服務： 如 `worker` 或資料庫初始化腳本，應僅掛載內部網絡。
- 外部服務： 僅有需要對外提供 HTTP 服務的 `web` 或 `flower` 才額外添加如 `traefik-public` 的外部網路。

**做法**： 在 `x-backend-common`中僅預設基礎網絡，特殊需求再於服務層級手動添加。

## 3. 單一真理來源
現在，如果你想新增一個全域環境變數，只需要改動最上方的 x-backend-common 一處，所有繼承該模板的服務都會同步生效。
這大幅降低了維護成本與出錯率。

# 四、 結語
使用 Anchors 與 Aliases 不僅是為了「少打幾個字」，更重要的是建立一個易於維護、邏輯清晰的部署架構。
對於需要運行多組 Celery Worker 或微服務的 Docker 環境來說，這是不可或缺的技巧。