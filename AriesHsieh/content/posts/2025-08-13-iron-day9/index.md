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

# CLLocationManager 權限管理

在使用 Core Location 時，首先需要處理用戶授權的問題。以下是基本步驟：

1. **初始化 CLLocationManager**：創建一個 CLLocationManager 的實例。
2. **請求權限**：根據需求請求 `whenInUse` 或 `always` 權限。
3. **處理授權狀態**：監聽授權狀態的變化，並根據用戶的選擇進行相應處理。

範例程式碼：

```swift
import CoreLocation

class LocationManager: NSObject, CLLocationManagerDelegate {
    private let locationManager = CLLocationManager()

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.requestWhenInUseAuthorization()
    }

    func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
        switch status {
        case .notDetermined:
            print("權限尚未決定")
        case .restricted, .denied:
            print("權限被拒絕")
        case .authorizedWhenInUse, .authorizedAlways:
            print("權限已授予")
        @unknown default:
            print("未知的授權狀態")
        }
    }
}
```

# 獲取用戶當前位置

在獲取用戶位置之前，請確保已經獲得授權。以下是基本步驟：

1. **啟動位置更新**：調用 `startUpdatingLocation` 方法。
2. **處理位置更新**：實現 `CLLocationManagerDelegate` 的 `didUpdateLocations` 方法來接收位置數據。

範例程式碼：

```swift
extension LocationManager {
    func startUpdatingLocation() {
        locationManager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        print("當前位置：\(location.coordinate.latitude), \(location.coordinate.longitude)")
    }
}
```

# 使用 Core Location

以下是在 SwiftUI 中整合 Core Location 的邏輯，如何取得並顯示用戶的當前位置。

## 範例程式碼

```swift
import SwiftUI
import CoreLocation

struct ContentView: View {
    @State private var userLocation: CLLocationCoordinate2D?
    private let locationManager = CLLocationManager()

    var body: some View {
        VStack {
            if let location = userLocation {
                Text("當前位置：\(location.latitude), \(location.longitude)")
                    .padding()
            } else {
                Text("正在獲取位置...")
                    .padding()
            }
        }
        .onAppear {
            setupLocationManager()
        }
    }

    private func setupLocationManager() {
        locationManager.delegate = LocationDelegate { location in
            self.userLocation = location.coordinate
        }
        locationManager.requestWhenInUseAuthorization()
        locationManager.startUpdatingLocation()
    }
}

class LocationDelegate: NSObject, CLLocationManagerDelegate {
    private let onUpdate: (CLLocation) -> Void

    init(onUpdate: @escaping (CLLocation) -> Void) {
        self.onUpdate = onUpdate
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        onUpdate(location)
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        print("獲取位置失敗: \(error.localizedDescription)")
    }
}

@main
struct CoreLocationApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

## 說明

1. **ContentView**：直接在 SwiftUI 視圖中處理 Core Location 的邏輯，並顯示用戶的當前位置。
2. **LocationDelegate**：用於處理 Core Location 的回調，將位置更新傳遞給視圖。

