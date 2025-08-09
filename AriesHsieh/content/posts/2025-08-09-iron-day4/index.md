---
title: "[Day 4] Swift 語言快速入門（一）"
date: 2025-08-09T01:41:46+08:00
draft: true
categories: ["iOS"]
tags: ["2025 iron", "SwiftUI"]
description: "Swift 語言基礎教學，適合初學者快速上手"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# Swift 語言基礎入門

本篇將帶大家快速認識 Swift 語言的基礎語法，適合完全沒有 iOS 或 Swift 經驗的初學者。內容包含：變數、常數、資料型別、型別推斷、型別安全與基本運算等。

## 實用工具

在學習新的語法時，我都喜歡自己實際操作驗證一次。Swift 可以使用 Apple 官方推出的 [Playground](https://www.apple.com/tw/swift/playgrounds/) 來執行 Swift 程式碼。

我這邊下載的是 macOS 版本，下載完畢打開 App 後，點擊左上方 File，選擇 New Book：

![playground1](playground1.png)

建立好新的 playground 後點擊進入，就可以在程式碼區域輸入你要執行的 code，按下右下角的 Run My Code，右側邊欄就會顯示執行結果。

![playground2](playground2.png)


## 變數與常數

在 Swift 中，使用 `var` 宣告變數（可變），使用 `let` 宣告常數（不可變）。

```swift
var age = 36        // 變數，可重新賦值
let name = "Aries"  // 常數，不可重新賦值
```

## 常見資料型別

Swift 是強型別語言，常見型別有：
- `String`：文字字串
- `Int`：整數
- `Double`：浮點數
- `Bool`：布林值（true/false）

```swift
var city: String = "Taipei"
var year: Int = 2025
var price: Double = 99.9
var isOpen: Bool = true
```

## 型別推斷與型別安全

Swift 會自動推斷型別，但仍然型別安全，不能將不同型別直接混用。

```swift
let score = 100         // 推斷為 Int
let pi = 3.14159        // 推斷為 Double
let isValid = false     // 推斷為 Bool
```

型別安全範例：
```swift
let number = 10
let text = "10"
// let result = number + text // 錯誤：Int 不能直接與 String 相加
// Binary operator '+' cannot be applied to operands of type 'Int' and 'String'
```

## 基本運算符號

Swift 支援常見運算：
- 加法：`+`
- 減法：`-`
- 乘法：`*`
- 除法：`/`
- 餘數：`%`

```swift
let a = 7
let b = 3
let sum = a + b      // 10
let diff = a - b     // 4
let prod = a * b     // 21
let div = a / b      // 2
let mod = a % b      // 1
```

## Optional 型別

Swift 有一個非常重要的語法特色：Optional 型別。這是 Swift 為了安全處理「值可能不存在」的情境而設計的。

### 為什麼需要 Optional？

在許多程式語言中，如果你嘗試存取不存在的值（例如 null、nil），常常會造成程式崩潰。Swift 為了避免這種錯誤，設計了 Optional 型別，讓你明確表示「這個變數可能有值，也可能沒有值」。

### Optional 的語法

在型別後面加上 `?`，就代表這個變數是 Optional 型別。

```swift
var nickname: String? = "Aries"
var petName: String? = nil
```
上例中，`nickname` 可能有值（"Aries"），也可能是 nil；`petName` 則目前沒有值。

### 如何取出 Optional 的值？

Optional 不能直接當作一般型別使用，必須「拆包」才能取得裡面的值。

#### 1. 強制拆包（不建議 XD）

```swift
var nickname: String? = "Aries"
print(nickname!) // Aries
```
如果 nickname 是 nil，強制拆包會造成程式崩潰。

![optional](optional.png)

#### 2. Optional Binding

```swift
var petName: String? = nil
if let name = petName {
    print("寵物名：\(name)")
} else {
    print("沒有寵物名")
}
```
這樣可以安全地判斷是否有值。

>至於這裡為什麼叫做 binding，因為在程式語言中，「binding」通常指的是「將一個值綁定到一個名稱（變數或常數）上」。在 optional binding 中，我們將 optional 內部的值（如果存在）綁定到一個新的非 optional 變數或常數上，也就是這裡的 `name`。

#### 3. Nil-Coalescing Operator

你也可以用事先給予預設值的方式來處理這個問題。

```swift
let input: String? = nil
let value = input ?? "預設值"
print(value) // 預設值
```

### Optional 在實務上的意義

Optional 讓程式更安全，強迫你思考「值可能不存在」的情境，減少因為 nil 造成的錯誤。這也是 Swift 強型別設計的理由之一。


剩下的明天待續～
