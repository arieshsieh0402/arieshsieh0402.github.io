---
title: "[Day 4] Swift 語言快速入門"
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

本篇將帶大家快速認識 Swift 語言的基礎語法，適合完全沒有 iOS 或 Swift 經驗的初學者。內容包含：變數、常數、資料型別、型別推斷、型別安全與基本運算。

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

## 流程控制

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
