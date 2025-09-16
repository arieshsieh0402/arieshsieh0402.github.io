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

今天要來講流程控制以及 class, sturct。這些是構成程式邏輯的基礎，理解它們，你就能讓 App 根據不同狀況做出反應。

Swift 提供多種流程控制語法，讓你根據條件或重複執行程式碼。

### 1. 條件判斷：if / else if / else

```swift
let score = 85
if score >= 90 {
    print("太棒了，A級！")
} else if score >= 60 {
    print("及格！繼續努力！")
} else {
    print("不及格！別灰心！")
}
```

你也可以用 && (AND) 和 || (OR) 來組合更複雜的條件：

```swift
let hasHomework = true
let isTired = false
if hasHomework && !isTired { // 必須完成作業「而且」不能累
    print("趕快寫作業！")
}
```

### 2. switch 判斷

Swift 的 switch 必須窮盡所有可能性（exhaustive），你不能漏掉任何一種情況，否則編譯器會報錯。這能幫我們避免很多 bug。

default 就是用來處理除了 case 以外所有情況。

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

3. 遍歷陣列：這是 for-in 最常見的用途，與多數程式語言大同小異。

```swift
let fruits = ["Apple", "Banana", "Cherry"]
for fruit in fruits {
    print("我喜歡吃 \(fruit)")
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

另外一個用法 repeat-while，它和 while 的差別是它保證至少會執行一次，因為它會先執行程式碼，才檢查條件。

```swift
var number = 10
repeat {
    print(number) // 即使條件一開始就是 false，這行還是會先執行一次
    number += 1
} while number < 5
// Console 會印出 10
```

## 函式（Function）

Swift 用 `func` 宣告函式。
一個完整的函式包含：函式名稱、參數（parameters）、回傳型別（return type）。

```swift
// name 是參數，-> String 代表這個函式會回傳一個字串
func greet(person name: String) -> String {
    return "Hello, \(name)!"
}

// 呼叫函式
let message = greet(person: "Aries")
print(message) // Hello, Aries!
```

person 稱為外部參數名（Argument Label），在呼叫函式時使用，增加可讀性。name 則是內部參數名（Parameter Name），在函式內部使用。如果不想使用外部參數名，可以用底線 _ 取代。

```swift
func add(_ a: Int, to b: Int) -> Int {
    return a + b
}
let sum = add(5, to: 10) // 呼叫時變得像一個句子
```

## 結構（Struct）

當你想把相關的資料打包在一起時，可以選擇 struct。

```swift
struct Person {
    var name: String
    var age: Int

    // 也可以在 struct 裡定義函式，稱為「方法」（Method）
    func introduce() {
        print("大家好，我叫 \(name)，今年 \(age) 歲。")
    }
}

// Swift 會自動幫我們生成一個初始化方法 (initializer)
let user = Person(name: "Aries", age: 36)
print(user.name) // Aries
user.introduce() // 大家好，我叫 Aries，今年 36 歲。
```



## 類別（Class）與結構（Struct）差異

Swift 有 class（類別）和 struct（結構）兩種型別，語法很像，但有重要差異：

1. class 是參考型別（Reference Type），struct 是值型別（Value Type）。
   - class 物件在傳遞時，會傳遞 reference，多個變數指向同一個物件。
   - struct 則是「複製」一份資料，彼此獨立。

2. 繼承（Inheritance）：class 可以繼承另一個 class 的屬性和方法，struct 不行。

3. 生命週期（Deinitializers）：class 可以有 deinit 方法，在物件被銷毀前執行清理工作，struct 沒有。

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

這是一個在 Swift 開發中非常重要的問題。Apple 官方的建議是：優先使用 struct。

- **struct**：當你的資料主要是用來封裝和傳遞時，就用 struct。例如：一個使用者設定、一個座標點、一個網路請求回傳的 Model。它更簡單、更安全，尤其在多執行緒環境下能避免很多奇怪的問題。（這也是為什麼 SwiftUI 的 View 幾乎都是 struct。）

- **class**：當你需要「共享狀態」或「繼承」時，才使用 class。例如：一個管理 App 全域狀態的物件。


## 本日小結

老實說，原本很猶豫要不要寫這一篇 Swift 語言基礎，畢竟現在有 AI 工具，查語法、範例、甚至直接生成程式碼都非常方便，很多人可能覺得「這種東西 Google 或 Copilot 一下就有了，何必再寫？」

但我認為，基礎知識還是有它的價值。AI 可以加速開發、解決問題，但如果開發者本身不懂底層原理或語法設計，遇到 bug 或特殊需求時，最後還是得回頭查，那不如一開始就先建立基本理解。

所以這篇文章，算是給初學者一個「打底」的機會，也給自己一個重新整理的過程。希望大家能在 AI 輔助下，依然保有對技術本質的好奇與追求。
