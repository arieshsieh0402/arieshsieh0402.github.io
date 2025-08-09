---
title: "[Day 6] SwiftUI 基礎元件與佈局實戰"
date: 2025-08-09T11:12:04+08:00
draft: true
categories: ["iOS"]
tags: ["2025 iron", "SwiftUI"]
description: "從零開始學習 SwiftUI 的基本元件與佈局技巧，並實作一個簡單的登入介面"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# SwiftUI 基礎元件介紹

在昨天中，我們學習了 Swift 語言的基礎語法。今天，我們要開始探索 SwiftUI 的基本元件和佈局方式。SwiftUI 採用宣告式語法，讓我們能夠更直覺地描述介面該長什麼樣子，而不是一步步告訴程式要如何建立它。

## 基本元件介紹

### Text：文字顯示
`Text` 是 SwiftUI 中最基本的文字顯示元件，可以加上各種修飾符來調整外觀：

```swift
Text("Hello, SwiftUI!")
    .font(.title)           // 設定字型大小
    .foregroundColor(.blue) // 設定文字顏色
    .bold()                 // 設定粗體
    .padding()             // 加上內距
```

### Image：圖片顯示
`Image` 可以顯示專案內的圖片資源或系統圖示：

```swift
// 顯示專案中的圖片
Image("logo")
    .resizable()           // 允許調整大小
    .scaledToFit()        // 保持比例縮放
    .frame(width: 100)    // 設定寬度

// 使用系統圖示
Image(systemName: "star.fill")
    .foregroundColor(.yellow)
```

### Button：按鈕元件
`Button` 能讓使用者進行互動，包含顯示內容和點擊動作：

```swift
Button(action: {
    // 點擊時執行的程式碼
    print("按鈕被點擊")
}) {
    // 按鈕的外觀
    Text("登入")
        .foregroundColor(.white)
        .padding()
        .background(Color.blue)
        .cornerRadius(8)
}
```

## 狀態管理：@State 與 @Binding

在 SwiftUI 中，我們使用特殊的屬性包裝器來管理元件的狀態：

### @State：管理元件內部狀態
當某個值會影響畫面顯示，且會隨著使用者操作而改變時，我們使用 `@State`：

```swift
struct CounterView: View {
    @State private var count = 0  // 宣告狀態變數

    var body: some View {
        VStack {
            Text("計數: \(count)")
            Button("增加") {
                count += 1  // 修改狀態，畫面會自動更新
            }
        }
    }
}
```

### @Binding：元件間的資料綁定
當需要在不同元件間共享和同步狀態時，使用 `@Binding`：

```swift
struct ToggleButton: View {
    @Binding var isOn: Bool  // 接收外部傳入的狀態

    var body: some View {
        Button(isOn ? "開啟" : "關閉") {
            isOn.toggle()    // 切換狀態
        }
    }
}

struct ParentView: View {
    @State private var switchState = false

    var body: some View {
        ToggleButton(isOn: $switchState)  // 傳遞狀態的綁定
    }
}
```

## 基本佈局容器

SwiftUI 提供了三種基本的堆疊容器來排列元件：

### VStack：垂直堆疊
```swift
VStack(spacing: 20) {  // spacing 設定元件間距
    Text("第一行")
    Text("第二行")
    Text("第三行")
}
```

### HStack：水平堆疊
```swift
HStack(alignment: .center) {  // alignment 設定對齊方式
    Image(systemName: "person")
    Text("使用者名稱")
}
```

### ZStack：重疊堆疊
```swift
ZStack {  // 後面的元件會疊在前面的上方
    Color.blue        // 背景
    Text("前景文字")   // 文字會顯示在藍色背景上
}
```

## 實戰練習：登入介面

讓我們運用上面學到的元件和佈局，製作一個簡單的登入介面：

```swift
struct LoginView: View {
    @State private var username = ""
    @State private var password = ""
    @State private var isLoading = false

    var body: some View {
        VStack(spacing: 20) {
            Text("歡迎登入")
                .font(.largeTitle)
                .bold()

            TextField("請輸入帳號", text: $username)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding(.horizontal)

            SecureField("請輸入密碼", text: $password)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding(.horizontal)

            Button(action: {
                isLoading = true
                // 模擬登入過程
                DispatchQueue.main.asyncAfter(deadline: .now() + 1.5) {
                    isLoading = false
                }
            }) {
                if isLoading {
                    ProgressView()
                        .progressViewStyle(CircularProgressViewStyle(tint: .white))
                } else {
                    Text("登入")
                }
            }
            .frame(width: 200)
            .padding()
            .background(Color.blue)
            .foregroundColor(.white)
            .cornerRadius(8)
        }
        .padding()
    }
}
```

這個登入介面包含了：
- 標題文字
- 帳號輸入框
- 密碼輸入框（使用 `SecureField` 確保密碼隱藏）
- 具有載入動畫的登入按鈕
- 適當的間距和樣式設定

## 小結

今天我們學習了 SwiftUI 的基本元件和佈局系統。SwiftUI 的宣告式語法讓我們能更直觀地描述想要的介面，而狀態管理機制則讓畫面能自動根據資料變化更新。雖然這些都是最基本的元件，但它們是構建更複雜介面的基礎。

在接下來的系列中，我們會繼續探索更多進階的 SwiftUI 功能，並逐步實作我們的里程標 App。

