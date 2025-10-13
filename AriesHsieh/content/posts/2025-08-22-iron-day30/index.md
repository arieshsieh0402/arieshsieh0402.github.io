---
title: "[Day 30] Finish line: 一段 SwiftUI 與 DevOps 之旅"
date: 2025-10-13T00:00:00+08:00
draft: false
categories: ["iOS"]
tags: ["2025 iron", "SwiftUI", "Azure", "DevOps"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

## 前言

三十天的鐵人賽，像一場技術馬拉松，終於來到了終點線前。最初的目標很明確：結合我對 iOS 開發的熱情與工作上學習到的 DevOps 實踐，從零到一打造一款名為「台灣公路 K 點通」的 SwiftUI App，並為它建立一套完整的自動化開發流程。

每天產出一篇文章是個挑戰，也是將腦中零散知識點串連成完整實踐地圖的過程。從動機、UI/UX 的草圖規劃，到 Swift 語法的基礎、SwiftUI 的畫面雕刻，再整合 Core Location 與 MapKit, Google map 的地圖服務，期間並導入 Azure Boards 進行專案管理，最後用 Azure Pipelines 實現 CI/CD。

## 旅程回顧：從痛點到 App Store 的完整路徑

這三十天的內容，可以劃分為四個階段，描繪了一個 iOS App 從概念誕生到最終上架的完整生命週期。

### 第一階段：從痛點到藍圖

萬事起頭難，在一開始專注於「想清楚要做什麼」與如何做。

我們從一個生活中的真實痛點出發：在公路上難以快速找到特定里程的確切位置，這個具體的目標，成為了整個專案的起點。在技術選型上，我們比較了 UIKit 與 SwiftUI，並基於開發效率、宣告式語法的優勢，以及想要嘗試新技術的心情，最終選擇了 SwiftUI 作為主力框架。

接著，我們在 Azure DevOps 上初始化了專案，設定了 Git Repo，並選擇了適合個人專案的 Basic 流程，為後續的專案管理與 CI/CD 建立基礎架構。

在開始動手寫 App 前，我們先透過使用者流程圖釐清了 App 的操作路徑，再用手繪 Wireframe 與 AI 工具輔助，描繪出 App 的畫面草圖，確保了後續開發的方向明確。

### 第二階段：掌握 SwiftUI

在這個階段建立 Swift 的語言基礎，快速複習了變數、常數、Optional 型別與流程控制等核心語法，並釐清了 Class 與 Struct 的關鍵差異。接著，我們深入 SwiftUI 的核心，從 Text、Image 與 Button 等基本元件著手，並學習運用 VStack、HStack 與 ZStack 這些佈局容器將畫面元素組合起來。在此過程中，我們掌握了 @State 與 @Binding 如何驅動畫面即時更新，理解了 SwiftUI 的宣告式思想。

為了實現核心的地圖功能，理解了如何使用 Core Location 框架來取得使用者位置，並藉由 MapKit 將地理資訊呈現在地圖上，再透過 Marker 和 Annotation 精準標示出目標點。

在互動細節上，實作了點擊圖釘後彈出 Sheet 顯示詳細資訊的功能，並利用 URL Scheme 串接 Apple Maps 與 Google Maps，提供給使用者導航服務。

另外，更在解決一個 Picker 元件顯示異常的 Bug 時，透過實際情境認識到 SwiftUI 的核心，也就是「View 是狀態的函數」。

### 第三階段：核心功能的實現

這階段實現 App 最關鍵的業務邏輯，里程資料的處理與地理圍欄通知。

我們從 App Bundle 中讀取 CSV 檔案，並將其順利解析成定義好的 struct。為了流暢地處理非同步流程，我們建立了一個名為 RoadDataManager 的 ObservableObject，讓 App 得以在背景讀取資料，並在完成後自動更新畫面。

接著，處理國道與省道迥異的里程格式。我們沒有選擇複製貼上程式碼，而是巧妙地運用了 Swift 的泛型（Generics）、閉包（Closure）與 KeyPath，設計出一個可重用的 findClosestMarker 函式。

為了讓 App 功能更加豐富，我們實作了地理圍欄（Geofencing）功能，使其能在背景持續監控使用者位置。當 App 重啟後，UI 狀態會隨之丟失。為此，我們利用 UserDefaults 與 Codable，將追蹤中的地點資訊進行持久化儲存，確保了使用者體驗在任何情況下都能保持一致。最後，為了確保功能如我們預想的一樣，使用 GPX 檔案模擬移動路徑，對地理圍欄的進入、離開及 App 重啟等各種情境，進行驗證。

### 第四階段：自動化

隨著 App 功能開發完成，我們的下一個目標，是將繁瑣且容易出錯的手動上架流程，用一套穩定可靠的自動化 CI/CD Pipeline 來取代。

我們利用 Apple 原生的 XCTest 框架，為核心的 RoadDataManager 撰寫了單元測試，以此確保資料解析邏輯的穩定性。接著，我們完成了行政上的前置作業，包含在 Apple Developer 網站註冊 App ID，以及在 App Store Connect 上建立 App 的基本資料。

完成繁瑣行政作業後，是 Pipeline 的實際建構。從申請 App Store Connect API 金鑰開始，按步驟設定好 Azure DevOps 的 Service Connection，並撰寫了 release-iOS.yaml 這份 pipeline，checkout 程式碼、安裝憑證、執行單元測試、打包簽署 .ipa 檔案，直到最終自動上傳至 Apple Store Connect。

在自動化的輔助下，未來可以輕鬆地將建置版本送上 Apple Store Connect，並提交審查，將重複、易錯的工作交給機器，讓開發者能夠專注於創造更有價值的核心功能。

## 結語

在過去，我可能認為 DevOps 是大型團隊的事情。但在工作上以及這次經驗讓我明白，即使是小團隊、個人專案，DevOps 的思維模式同樣重要。從使用 Azure Boards 規劃任務開始，到用 CI/CD 自動化部署，整個開發過程會變得清晰。每一次的 commit 都有對應的 work item，每一個發布的版本都有清晰的 Git Tag，開發流程是非常清楚的。

學習單一技術（如 SwiftUI 或 Core Location）與打造一個完整的 App，是完全不同的體驗。一個真實的專案，會強迫你用不同角度去思考所有環節：從 UI/UX、資料處理、狀態管理、錯誤處理，到測試、部署與上架規範。遇到問題，就會強迫你想辦法去解決。

感謝 iThome 提供這個機會，讓我有機會完成這場自我挑戰。
