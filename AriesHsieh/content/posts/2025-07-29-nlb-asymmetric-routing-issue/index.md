---
title: "網路負載平衡中的「非對稱路由」問題"
date: 2025-07-29T16:59:23+08:00
draft: false
categories: ["網路架構"]
tags: ["Load balancer", "Asymmetric Routing", "Subnet", "Gateway", "DNAT", "SNAT", "TCP/IP"]
description: "探討網路負載平衡中因同網段部署造成的非對稱路由問題與解決方案"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

某支 API 要掛載到 APIM 上，我的同事問我這支 API 的後端伺服器有跟 Data plane 同網段嗎？同網段可能會有問題喔～
我又嗅到新知識的味道了，趕緊來研究跟記錄一下：

# TCP 4-Tuple

首先必須先簡單提一下這個概念。這概念是指，每個 TCP 連線都是透過四個關鍵資訊組合成唯一識別，分別為：

```
(Client IP, Client Port, Server IP, Server Port)
```

1. Client IP: 來源 IP 位址
2. Client Port: 來源埠號
3. Server IP: 目的地 IP 位址
4. Server Port: 目的地埠號

>或稱 Source & Destination
這四個資訊合起來，可確保網路上每一條 TCP 連線都是獨一無二的。

因此若 TCP 三方交握時，如果接收到的封包其 4-tuple 不符合系統 listen 的條件（如目的 IP 或目的 Port 不正確），該封包會被丟棄，不會產生連線。

# 情境分析

## 不同網段（非對稱路由）

假設：
- Client：10.1.1.10 (網段 A)
- NLB VIP：10.2.2.100 (網段 B)
- Server A：10.2.2.50 (網段 B)

- 請求階段（去程）
1. Client → Gateway：
    - Client 準備發送一個封包給 NLB VIP (10.2.2.100)。
    - 它檢查自己的路由表，發現 10.2.2.100 不在本地網段 A，故必須將封包交給預設閘道 (Default Gateway) 來轉送。
    - 此時封包內容：
        - 來源 IP：10.1.1.10
        - 目的 IP：10.2.2.100 (VIP)
2. Gateway → NLB → Server A：
    - 封包經由路由器，最終到達 NLB。
    - NLB 執行 DNAT (目的位址轉換)，將目的 IP 從 VIP (10.2.2.100) 改為 Server A (10.2.2.50)。
    - 此時封包內容更新為：
        - 來源 IP：10.1.1.10 (不變)
        - 目的 IP：10.2.2.50 (Server A)
    - NLB 將此封包傳給 Server A。

- 回應階段 (回程)
1. Server A → Gateway：
    - Server A 收到請求，準備回覆給 Client(10.1.1.10)。
    - 它檢查自己的路由表，發現 10.1.1.10 不在本地網段。因此，它也必須將回應封包交給自己的預設閘道。在這種架構下，這個閘道器通常就是負載平衡器本身，或是指向它的路由器。
    - 此時回應封包內容：
        - 來源 IP：10.2.2.50 (Server A)
        - 目的 IP：10.1.1.10 (Client)
2. Gateway (NLB) 進行反向轉換：
    - NLB 收到了來自 Server A 的回程封包。
    - NLB 查詢自己的連線狀態表 (State Table)，發現這個封包屬於先前建立的一個連線。
    - 它執行反向來源位址轉換 (Reverse SNAT)，將封包的來源 IP 從 Server A (10.2.2.50) 改回 VIP(10.2.2.100)。
    - 封包內容最終更新為：
        - 來源 IP：10.2.2.100 (VIP)
        - 目的 IP：10.1.1.10 (Client)
3. NLB → Client：
    - NLB 將這個修改過的封包送回給 Client。
    - Client 收到一個來自 10.2.2.100 (VIP) 的回應。因為它當初就是向 VIP 發起連線，所以 TCP 4-Tuple 可對應，連線視為合法。

## 同個網段（對稱路由）

假設：
- Client：10.2.2.10 (網段 B)
- NLB VIP：10.2.2.100 (網段 B)
- Server A：10.2.2.50 (網段 B)

1. 請求階段（去程）
  - Client → NLB：
    - Client 準備發送封包給 NLB VIP (10.2.2.100)。
    - 它檢查自己的路由表，發現 10.2.2.100 在同一個網段內，因此直接透過 Layer 2 (MAC 位址) 傳送，不需要經過閘道器。
    - 此時封包內容：
        - 來源 IP：10.2.2.10
        - 目的 IP：10.2.2.100 (VIP)
2. NLB → Server A：
    - NLB 執行 DNAT，將目的 IP 從 VIP (10.2.2.100) 改為 Server A (10.2.2.50)。
    - 此時封包內容更新為：
        - 來源 IP：10.2.2.10 (不變)
        - 目的 IP：10.2.2.50 (Server A)
    - NLB 將此封包傳給 Server A。
3. 回應階段（回程）- 問題發生處
    - Server A 收到請求後，準備回覆給 Client(10.2.2.10)。
    - Server A 檢查自己的路由表，根據路由表的最長前綴匹配原則（Longest Prefix Match）[^1]，發現 10.2.2.10 在同一個網段內。因此會直接在同一個 Layer 2 網段內回應 Client，繞過了 NLB。
3. 直接回應（繞過 NLB）：
    - Server A 直接將回應封包傳送給 Client，完全繞過了 NLB。
    - 此時回應封包內容：
        - 來源 IP：10.2.2.50 (Server A)
        - 目的 IP：10.2.2.10 (Client)
4. TCP 4-Tuple 不匹配：
    - Client 收到來自 10.2.2.50 的回應封包。
    - 但是 Client 原本是向 10.2.2.100 (VIP) 發起連線的。
    - TCP 4-Tuple 不匹配：
        - 預期的 4-Tuple：(10.2.2.10, port_x, 10.2.2.100, port_y)
        - 實際收到的 4-Tuple：(10.2.2.10, port_x, 10.2.2.50, port_y)
    - Client 無法將此回應與原本的連線建立關聯，因此會丟棄此封包。


# 解決方案
1. 啟用 SNAT (Source NAT)[^2]：
   NLB 同時執行 DNAT 和 SNAT，將來源 IP 也改為 NLB 的 IP，強制回程流量必須經過 NLB
2. 分離網段：
   最直接的做法，將 Client 和 Server 放在不同網段。

[^1]: https://lin0204.blogspot.com/2016/08/prefix.html
[^2]: https://devops.stackexchange.com/questions/1298/application-calling-aws-internal-load-balancer-in-same-subnet-is-timing-out
