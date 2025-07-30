---
title: "高可用性叢集(HA cluster)運作模式 - 以 Kong CP 為例"
date: 2025-07-25T22:05:42+08:00
draft: false
categories: ["api-gateway", "database"]
tags: ["kong", "PostgreSQL", "HA cluster"]
description: "簡述 HA cluster Active-Passive 和 Active-Active 模式"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

## 前言

某天我在公司 Kong 測試環境的 control plane 上作業時，才發現其中一台 control plane 的服務竟然沒啟用。好奇之下問了之前負責維護的同事，結果他說：「本來就只會啟用一台啊！」於是我又多學了一課，身為啥都學來者不拒的工程師，當然要記錄下來。

## HA cluster 高可用性叢集

根據 wikipedia 的定義：「高可用性叢集(High-availability clusters)是以最短的中斷時間為目標而可靠地運作的，支撐伺服器應用的一組電腦。它們通過使用高可用性軟體來管理叢集中的冗餘電腦，當系統組件出現故障時，這些電腦可以繼續提供服務。」簡單來說，就是有備援主機能在其中一台掛掉時接手，讓服務不中斷。

在接手 Kong 之前，我手上的系統也幾乎都是一個服務同時掛載在由兩個節點組成的 cluster 上，流量會透過 NLB(Network Load Balancer，網路負載平衡器)分流到兩個 node 上。

HA cluster 常見有兩種運作模式：Active-Passive 和 Active-Active。

- Active-Passive 模式：一台節點（主節點）負責運行服務並處理所有流量，另一台節點則處於待命（備援）狀態，只有主節點故障時才會接手服務。
- Active-Active 模式：兩台或多台節點同時運行服務，互相備援，平時都在積極處理流量。當其中一台節點故障時，其他節點會接手其服務。

所以，這次 control plane 採用的是 Active-Passive 模式。But why? 一定有什麼原因吧？（後來想起其實同事很久以前有說過，但我忘了，且當時也沒說那麼細，所以說做筆記很重要）。

## Control plane v PostgreSQL

Kong 的 Control Plane 負責集中管理 API Gateway 的所有設定、認證，以及 policy 和各種 plugins 的管理操作。不過 control plane 本身並不保存資料，只是管理和派送設定任務。所有設定與狀態資料都必須儲存在獨立運行的 PostgreSQL 資料庫裡，control plane 會以資料庫客戶端身份，透過網路連線到 PostgreSQL 資料庫伺服器，每次有設定變更、查詢、同步請求時都會讀寫這個資料庫。

根據官方文件[^1]，PostgreSQL 不支援多台主機或多個資料庫 instance 同時掛載、讀寫同一份 data directory，否則可能會有資料毀損的風險。事實上，有人實驗過[^2]，即使強行繞過 PostgreSQL 的檢查機制，在你啟用另一個 instance 時，原本掛載的 instance 也會被強制斷開。

所以，這就是我們架構設計下，同時只會啟用一個 control plane 服務的原因啦～


[^1]: https://wiki.postgresql.org/wiki/Shared_Storage
[^2]: https://www.dbi-services.com/blog/can-you-start-two-or-more-postgresql-instances-against-the-same-data-directory/
