---
title: "[Day 9] Core Location 基礎"
date: 2025-08-13T13:22:43+08:00
draft: true
categories: ["iOS"]
tags: ["Swift", "SwiftUI", "iOS", "Core Location"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

Core Location 是 iOS 開發中用於處理地理位置相關功能的框架。今天的目標是了解如何使用 Core Location 來管理權限以及獲取用戶的當前位置。

# CLLocationManager 解析

## 授權

在使用 Core Location 時，首先需要處理用戶授權的問題，這也是我認為 iOS 開發有一點小複雜（惱人）的地方。

首先我們先在專案資料夾建立一個新的 .swift 檔 `LoactionManager.swift`。

![addFile](addFile.png)

可以在專案資料夾按快捷鍵 `cmd + N`，選擇 iOS -> Swift File。

我們的目的是建立一個獨立的管理者 LocationManager，它會處理向使用者請求定位權限、接收座標更新、並將這些資訊即時「發布」給 SwiftUI 介面，讓畫面可以根據最新的位置或權限狀態自動更新。

```swift
import CoreLocation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {
    // ....
}
```

- NSObject: 繼承自 NSObject。因為 Core Location 框架是基於早期 Objective-C 的設計模式。

- ObservableObject: 遵循 ObservableObject 協定。這是一個來自 Combine 框架的宣告，表示這個物件可以被 SwiftUI 的 View 所「觀察」。一旦物件內被 `@Published` 標記的屬性發生改變，它會自動通知所有正在觀察它的 View 進行更新。

- CLLocationManagerDelegate: 遵守 CLLocationManagerDelegate 協定。這表示我們的 LocationManager 類別有能力接收並處理來自 CLLocationManager 的各種事件回調，例如「權限狀態改變」或「位置更新」。


