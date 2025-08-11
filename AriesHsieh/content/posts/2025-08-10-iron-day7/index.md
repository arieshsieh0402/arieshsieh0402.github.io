---
title: "[Day 7] SwiftUI 列表與導航"
date: 2025-08-10T19:02:28+08:00
draft: false
categories: ["iOS"]
tags: ["Swift", "SwiftUI", "iOS", "List", "Navigation"]
description: "本文介紹 SwiftUI 中的 List 元件與 ForEach 的使用方法，以及如何透過 NavigationView 和 NavigationLink 實現畫面導航與資料傳遞。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# SwiftUI 列表與導航：打造動態資料展示與畫面切換

在 SwiftUI 的開發過程中，列表（List）和導航（Navigation）是兩個非常重要的基礎元件。它們不僅能幫助我們有效地展示大量資料，還能提供直覺的畫面切換機制。今天我們將深入探討這兩個重要的主題，並透過實作一個簡單的產品列表應用來理解相關概念。

## List 元件與 ForEach 的基本使用

SwiftUI 的 List 元件提供了一個簡單且高效的方式來展示資料集合。我們先來看看如何建立一個基本的列表：

```swift
struct ContentView: View {
    let items = ["項目 1", "項目 2", "項目 3"]

    var body: some View {
        List {
            ForEach(items, id: \.self) { item in
                Text(item)
            }
        }
    }
}
```

在上面的例子中，我們使用了 `ForEach` 來遍歷陣列中的元素。`id: \.self` 告訴 SwiftUI 使用項目本身作為唯一識別符。

## Identifiable 協定的應用

為了更好地管理列表中的資料，我們可以讓我們的資料模型遵循 `Identifiable` 協定：

```swift
struct Product: Identifiable {
    let id = UUID()
    var name: String
    var price: Double
    var description: String
}

struct ProductListView: View {
    let products = [
        Product(name: "iPhone 14", price: 27900, description: "最新款 iPhone"),
        Product(name: "MacBook Air", price: 35900, description: "M2 晶片筆電"),
        Product(name: "iPad Pro", price: 27900, description: "專業平板電腦")
    ]

    var body: some View {
        List(products) { product in
            VStack(alignment: .leading) {
                Text(product.name)
                    .font(.headline)
                Text("NT$ \(Int(product.price))")
                    .foregroundColor(.gray)
            }
        }
    }
}
```

## 實現畫面導航

接下來，我們將使用 `NavigationView` 和 `NavigationLink` 來實現產品列表到詳細資訊的導航：

```swift
struct ProductDetailView: View {
    let product: Product

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            Text(product.name)
                .font(.title)
            Text("NT$ \(Int(product.price))")
                .font(.headline)
                .foregroundColor(.blue)
            Text(product.description)
                .font(.body)
            Spacer()
        }
        .padding()
        .navigationBarTitleDisplayMode(.inline)
    }
}

struct ProductListView: View {
    let products = [
        Product(name: "iPhone 14", price: 27900, description: "最新款 iPhone"),
        Product(name: "MacBook Air", price: 35900, description: "M2 晶片筆電"),
        Product(name: "iPad Pro", price: 27900, description: "專業平板電腦")
    ]

    var body: some View {
        NavigationView {
            List(products) { product in
                NavigationLink(destination: ProductDetailView(product: product)) {
                    VStack(alignment: .leading) {
                        Text(product.name)
                            .font(.headline)
                        Text("NT$ \(Int(product.price))")
                            .foregroundColor(.gray)
                    }
                }
            }
            .navigationTitle("產品列表")
        }
    }
}
```

## 重要觀念總結

1. **List 與 ForEach**
   - List 提供垂直捲動的列表介面
   - ForEach 用於遍歷集合資料
   - 需要提供唯一識別符（id）

2. **Identifiable 協定**
   - 為資料模型提供唯一識別符
   - 簡化 List 和 ForEach 的使用
   - 透過 UUID() 自動生成唯一 ID

3. **Navigation 導航**
   - NavigationView 作為導航容器
   - NavigationLink 處理導航邏輯
   - 支援資料傳遞和標題設定

## 小結

今天我們學習了 SwiftUI 中的列表和導航功能，這兩個元件在 iOS 應用開發中扮演著重要角色。透過實際範例，我們了解了如何建立動態列表、管理資料模型，以及實現畫面導航。這些基礎知識將幫助我們開發出更豐富的使用者介面。

下一篇文章，我們將探討更進階的主題。如果您對今天的內容有任何問題，歡迎在下方留言討論。
