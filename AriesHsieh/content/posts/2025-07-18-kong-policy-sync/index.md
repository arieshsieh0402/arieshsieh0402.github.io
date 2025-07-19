---
title: "簡述 Kong API Gateway Control Plane 與 Data Plane 設置同步機制"
date: 2025-07-18T14:19:32+08:00
draft: false
categories: ["api-gateway"]
tags: ["kong"]
description: "紀錄釐清在 kong hybrid mode 下，control plane 與 data plane 之間 sync 設置的方向"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

![pic](https://images.unsplash.com/photo-1752353739067-357d9ff65d4f?q=80&w=2084&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D)

***我喜歡在文章裡面放張圖，雖然現在 AI 生圖很方便，但還是喜歡放自然一點的，不論是自己拍的或是網路上看到的(from unsplash)***

---

## 前言

在 Kong API Gateway 的日常維運中，Control plane 與 Data plane 的同步機制是個蠻值得關注的概念。最近在檢視公司架構圖時，我發現到 Control plane 與 Data plane 之間的 policy sync 方向標示為 Data plane 向 Control plane「拉取」最新設置。

![sync](2025-07-18-kong-policy-sync.jpeg)

>簡單畫，公司實際架構圖沒這麼醜

在看到公司架構圖標示 Data plane 會「拉取」設定時，其實我一開始沒有多想，且防火牆確實只開了 Data plane 到 Control plane 的 8005 port。
但後來仔細思考，覺得這種設計有點奇怪——如果是 Data plane 不斷輪詢 Control plane 是否有設定變更，那輪詢頻率要怎麼抓？萬一設定異動很頻繁，會不會造成不必要的流量和效能浪費？而且從系統設計的角度來看，應該是由管理端（Control plane）主動通知執行端（Data plane）有異動才比較合理。這讓我開始思考，究竟同步機制的實際運作方式是什麼？

這些疑問促使我去查官方文件，才發現實際上是 Control plane 主動推送設定(如下圖)，和我原本的認知有落差。

![kong-sync](2025-07-18-kong-policy-sync-2.png)

## Control plane 與 Data plane

先簡單說明一下這兩個東西是什麼。

Kong Gateway 在 hybrid mode 下，有兩個重要的角色，分別為 Controle plane 與 Data plane，其功能分別為：

- **Control plane**: 負責管理與設定整個系統的節點。提供設定管理介面，你可以在這裡新增、修改 route、service、plugin 等設定，這些設定會被下發給其他節點（如 Data Plane）。
- **Data plane**: 實際處理用戶流量的節點，這些節點負責根據 Control Plane 下發的設定去代理、轉發 API 請求。

## Control plane 與 Data plane 之間的通訊機制

根據官方文件說明：

>When you create a new Data Plane node, it establishes a connection to the Control Plane. The Control Plane listens on port 8005 (Kong Gateway) or 443 (Konnect) for connections and tracks any incoming data from its Data Planes.
>
>Once connected, every API or Kong Manager/Konnect UI action on the Control Plane triggers an update to the Data Planes in the cluster.

也就是說，公司的架構圖只對了一半。最初是由 Data plane 主動與 Control plane 建立連線沒錯，但連線後，是由 Control plane 向 Data plane 推送設定。

詳細的流程圖如下：

```mermaid
sequenceDiagram
participant DP as Data Plane
participant CP as Control Plane

DP->>CP: 發起 WebSocket 連線 (port 8005)
CP->>DP: mTLS handshake
CP->>DP: 推送完整配置 (所有 workspace)
DP->>CP: 每 30 秒發送 heartbeat
CP->>DP: 設置變更時推送更新
```

## 結語

Kong Gateway 在 hybrid mode 下，Data plane 雖然主動連線到 Control plane，但實際的設定同步是由 Control plane 主動推送，確保所有 Data plane 節點都能即時獲得最新的設置。也讓我釐清了原本的認知落差，官方文件還是很重要的～

---

REF:

1. https://developer.konghq.com/gateway/hybrid-mode/
2. https://developer.konghq.com/gateway/cp-dp-communication/
3. https://support.konghq.com/support/s/article/How-to-debug-the-Control-Plane-DataPlane-web-socket-communication
