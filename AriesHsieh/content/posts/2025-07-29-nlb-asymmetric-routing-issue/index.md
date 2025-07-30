---
title: "網路負載平衡中的「非對稱路由」問題"
date: 2025-07-29T16:59:23+08:00
draft: true
categories: ["網路架構"]
tags: ["Load balancer", "Asymmetric Routing", "Subnet", "Gateway", "DNAT"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

某支 API 要掛載到 APIM 上，我的同事問我這支 API 的後端伺服器有跟 Data plane 同網段嗎？同網段可能會有問題喔～
我又嗅到新知識的味道了，趕緊來研究跟記錄一下：

# TCP 4-Tuple

首先必須先提一下這個概念。

# 同個網段（對稱路由）
# 不同網段（非對稱路由）

