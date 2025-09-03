---
title: "[Day 24] 地理圍欄通知（一）"
date: 2025-08-22T10:55:40+08:00
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

首先簡要回顧前幾天的進度：

- Day 11：我們實作了地理圍欄的核心功能，讓 App 能夠在用戶進入或離開特定區域時觸發事件 。

- Day 21 & 22：我們優化了地圖互動，實現了點擊圖釘後彈出詳細資訊視窗（Sheet），並加入了跳轉至 Apple Maps 或 Google Maps 的功能 。

- Day 23：我們導入了 UserDefaults 和 Codable，實現了追蹤狀態的持久化。現在 App 即使關閉重啟，也能記得正在追蹤的地點 。

但程式碼跑起來是一回事，但在真實世界中是否可運作又是另一回事，所以今天的核心任務是驗證。我們將設計一系列的測試情境，徹底檢驗地理圍欄通知的穩定性與準確性，並記錄測試過程與結果。

# 測試準備

## 使用 GPX 檔進行定位模擬

在 Day 11 測試地理圍欄功能時，我們是使用修改模擬器定位的方式來測試。今天我們用另外一種方式，即在 Xcode 匯入 GPX 檔用來模擬「進入區域」、「離開區域」的行為。

你可以在 Xcode 裡面新增檔案，直接新增 GPX 檔:

```xml
<?xml version="1.0"?>
<gpx version="1.1" creator="Xcode">

    <!--
     Provide one or more waypoints containing a latitude/longitude pair. If you provide one
     waypoint, Xcode will simulate that specific location. If you provide multiple waypoints,
     Xcode will simulate a route visiting each waypoint.
     -->
    <wpt lat="37.331705" lon="-122.030237">
        <name>Cupertino</name>

        <!--
         Optionally provide a time element for each waypoint. Xcode will interpolate movement
         at a rate of speed based on the time elapsed between each waypoint. If you do not provide
         a time element, then Xcode will use a fixed rate of speed.

         Waypoints must be sorted by time in ascending order.
         -->
        <time>2014-09-24T14:55:37Z</time>
    </wpt>

</gpx>
```

這是蘋果 GPX 檔的模板，可以直接修改 `<wpt>` 標籤中的 lat（緯度）和 lon（經度）數值，將其變更為需要的任何地點。

這裡我們不自己加入地點，使用第三方工具來產出 GPX 檔案會比較快。

我們先找出要追蹤定位點的位置，例如我們先用 App 找出「台 9 線 3k」的位置，並且用 Google Map 先確認位置：

![alt text](googleMap.png)

接著，開啟[Gpxgenerator](https://www.gpxgenerator.com)這個網站，可以在定位點的前後新增 waypoint 以模擬行徑路線：


![alt text](gpxgenerator.png)

下方可以選擇模擬的速度，選定一個適當的速度後，點選 Download GPX，並將檔案加入 Xcode 專案裡。

