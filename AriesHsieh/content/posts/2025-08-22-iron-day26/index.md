---
title: "[Day 26] 實現 iOS 自動化部署 - 設定 Azure Pipelines"
date: 2025-08-22T10:55:46+08:00
draft: true
categories: []
tags: []
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

來到第 26 天，App 核心功能大致上已開發告一段落，是時候建立一套自動化的 CI/CD 流程了。目標是當我將程式碼推送到一個特定的 Git Tag 時，Azure DevOps 能自動幫我完成測試、建置、簽署，並將 App 上傳到 TestFlight，進行測試與準備提交審查。

過去我雖然有在企業環境使用 Azure Pipelines 的經驗，但那是針對企業內部憑證的發布流程，這次，是直接上架到 App Store 的標準流程，算是新的嘗試。










https://ithelp.ithome.com.tw/articles/10268594

