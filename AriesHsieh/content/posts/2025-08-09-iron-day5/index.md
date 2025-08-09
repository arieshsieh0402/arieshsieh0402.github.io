---
title: "[Day 5] Swift 語言快速入門（二）"
date: 2025-08-09T10:59:53+08:00
draft: true
categories: ["iOS"]
tags: ["2025 iron", "SwiftUI"]
description: "Swift 語言基礎教學，適合初學者快速上手"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

## 流程控制

今天要講流程控制以及 class, sturct。

Swift 提供多種流程控制語法，讓你根據條件或重複執行程式碼。

### 1. 條件判斷：if / else

```swift
let score = 85
if score >= 60 {
    print("及格！")
} else {
    print("不及格！")
}
```

### 2. switch 判斷

```swift
let animal = "dog"
switch animal {
case "cat":
    print("貓貓")
case "dog":
    print("狗狗")
default:
    print("其他動物")
}
```

### 3. 迴圈 for-in

Swift 的 for-in 迴圈可以用兩種範圍語法：

1. 閉區間（`...`）：包含起始與結束值。

```swift
for i in 1...5 {
    print(i) // 1, 2, 3, 4, 5
}
```

2. 半開區間（`..<`）：包含起始值，不包含結束值。

```swift
for i in 1..<5 {
    print(i) // 1, 2, 3, 4
}
```

### 4. while 迴圈

```swift
var count = 3
while count > 0 {
    print(count)
    count -= 1
}
```

## 函式（Function）

Swift 用 `func` 宣告函式。

```swift
func greet(name: String) -> String {
    return "Hello, \(name)!"
}

let message = greet(name: "Aries")
print(message) // Hello, Aries!
```

## 結構（Struct）

Swift 可以用 `struct` 來定義資料結構。

```swift
struct Person {
    var name: String
    var age: Int
}

let user = Person(name: "Aries", age: 36)
print(user.name) // Aries
```

## 類別（Class）與結構（Struct）差異

Swift 有 class（類別）和 struct（結構）兩種型別，語法很像，但有重要差異：

1. class 是參考型別（Reference Type），struct 是值型別（Value Type）。
   - class 物件在傳遞時，會傳遞 reference，多個變數指向同一個物件。
   - struct 則是「複製」一份資料，彼此獨立。

2. class 可以繼承（inheritance），struct 不行。

3. class 可以有 deinit，struct 沒有。

### 範例比較

```swift
class Dog {
    var name: String
    init(name: String) {
        self.name = name
    }
}

var dog1 = Dog(name: "小黑")
var dog2 = dog1
dog2.name = "小白"
print(dog1.name) // 小白（同一物件）

struct Cat {
    var name: String
}

var cat1 = Cat(name: "咪咪")
var cat2 = cat1
cat2.name = "花花"
print(cat1.name) // 咪咪（彼此獨立）
```

### 實務上怎麼選？
- **struct**：用於資料結構、Model、簡單資料封裝（SwiftUI 幾乎都用 struct）。
- **class**：用於需要繼承、物件共享、管理狀態的場景（如 UIKit、複雜邏輯）。


## 小結

老實說，原本很猶豫要不要寫這一篇 Swift 語言基礎，畢竟現在有 AI 工具，查語法、範例、甚至直接生成程式碼都非常方便，很多人可能覺得「這種東西 Google 或 Copilot 一下就有了，何必再寫？」

但我認為，基礎知識還是有它的價值。AI 可以加速開發、解決問題，但如果開發者本身不懂底層原理或語法設計，遇到 bug 或特殊需求時，最後還是得回頭查，那不如一開始就先建立基本理解。

所以這篇文章，算是給初學者一個「打底」的機會，也給自己一個重新整理的過程。希望大家能在 AI 輔助下，依然保有對技術本質的好奇與追求。
