---
title: "[Day 28] 實現 iOS 自動化部署（三）- 建構完整的 Pipeline"
date: 2025-08-22T10:56:09+08:00
draft: true
categories: []
tags: []
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

## 為我們的 App 加上 logo

在我們開始寫我們的 pipeline 之前，我突然想到有一件很重要的事情，那就是為我們的 App 加上一個看起來很讚的 icon。App icon 是使用者在 App Store 的第一個接觸點，某種程度上算是決定使用者有沒有興趣的關鍵。

蘋果對 icon 的格式有明確的要求，必須是一個 1024x1024 像素、方形、無圓角、無透明度的 PNG 檔案。當你圖片檔拖入 Xcode 的 Assets.xcassets 中對應的 AppIcon 位置時，Xcode 會自動根據這個圖檔，生成所有需要的尺寸，以適應 iPhone、iPad、通知、Spotlight 搜尋、設定頁面等各種顯示情境。

![alt text](image.png)

>你也可以按照亮色或暗色主題提供不同的 icon 樣式。

---

接著我們要進入一個十分重要的部分。

## 蘋果的 Code sign 機制

### 為什麼要有這個機制？

蘋果希望確保用戶從 App Store 下載或安裝的每一個 App，都來自於「已知的開發者」，且「未被竄改」。為了實現這個目標，蘋果建立了一套基於數位簽章的信任鏈。

為了達到這個目的，有以下幾個核心概念：

1. 開發者憑證 (Certificate)

可以把它憑證想像為蘋果官方頒發給開發者的身份證，用來證明「該開發者為該開發者本身」。它保證了 App 是由一個已知的、經過蘋果驗證的開發者所建立的。
而憑證又可區分為開發憑證與分發憑證：

- 開發憑證 (Development Certificate): 此張憑證讓開發者可以在自己的測試裝置上安裝和測試 App。
- 分發憑證 (Distribution Certificate): 只有用此憑證簽署的 App，才能上傳到 App Store。

2. App ID
每個 App 具有獨一無二的的全球唯一識別碼，用來特定 App。

3. 描述檔 (Provisioning Profile)

它像一張詳細的通行證，將上述所有資訊綁定在一起，代表了以下事情：

- 誰 (Who)：哪個開發者（憑證）可以簽署這個 App？

- 做什麼 (What)：這個 App（App ID）可以使用哪些服務（如推播通知、iCloud）？

- 在哪裡 (Where)：這個 App 可以在哪些裝置上運行？
    - 開發階段: 一個包含特定裝置 UDID 的列表。
    - 發布階段: App Store 的所有裝置。

當使用者嘗試在裝置上運行 App 時，iOS 會檢查描述檔，確認所有資訊都正確無誤後才能執行。

### 如何實現這個機制？我們該怎麼做？

現在我們了解了「為什麼」，接著來看「如何實現」。

- CSR (Certificate Signing Request)

你需要先在你的 Mac 的鑰匙圈存取 (Keychain Access) 中產生一個 CSR。這個過程會在您的 Mac 上創建一對公私鑰，並把包含公鑰的 CSR 給蘋果。

蘋果收到 CSR 後，會用他們的根憑證為你的公鑰簽章，並產出一份數位憑證 (.cer)。下載這個憑證並安裝到鑰匙圈後，它會自動與你本機的私鑰配對，往後只有您的這台 Mac 才能「簽署」App。

### 本機與 CI/CD 環境的差異

若單純在本機的 Xcode 編譯專案，產出 .ipa 檔並上傳到 App Store Connect，只要你在 Xcode 中勾選「Automatically manage signing」並登入 Apple ID，Xcode 會自動完成 CSR 生成、上傳、下載憑證、建立描述檔等所有繁瑣步驟。而因為是本機編譯，所有憑證和私鑰都存放在你開發 Mac 的鑰匙圈中隨時可用。

![alt text](image-1.png)

因此在 Xcode 中要上架 App，Xcode 幫你處理了所有這些繁雜工作，我們只要「Archive」和「Distribute」就好，十分單純。

然而，透過外部 CI/CD 工具的情況就大不相同了，

Azure Pipeline 的虛擬機是一個乾淨、無記憶的環境。它不知道你的 Apple ID，也沒有儲存您的私鑰。因此，Xcode 的「自動簽章」在這裡無法運作。

你必須手動告訴這個新環境如何完成簽章，包含：

1. 提供私鑰與憑證: 你必須從 Mac 中匯出包含私鑰的 .p12 檔案，並將存放在 Azure DevOps 的 Secure Files 中。

2. 提供描述檔: 你必須手動從 Apple Developer Portal 下載 App Store 用的 .mobileprovision 檔案，並同樣上傳到 Secure Files。

3. 在 Pipeline 中安裝: 在 YAML 檔案中，您需要使用一系列任務，在每次建置時將這些檔案安裝到虛擬機的臨時鑰匙圈中。

---

## 準備 Apple Code sign 相關檔案

### 產出 CSR 並匯入憑證

首先，打開你 Mac 上的「鑰匙圈存取(Keychain Access)」應用程式。檢查一下你的憑證列表，通常你會看到已有的開發憑證，但可能還沒有用於 App Store 上架的分發憑證。

![alt text](keychain1.png)

點擊螢幕左上角的選單列「Keychain Access」> 「Certificate Assistant」> 「Request a Certificate From a Certificate Authority...」

![alt text](image-3.png)

在彈出的視窗中，輸入你的 Email 和名字（建議與 Apple 開發者帳號一致），然後勾選「Saved to disk」選項，點擊 Continue。系統將會在你指定的位置產出一份 CSR 檔案。

![alt text](image-4.png)

接著到 Apple Developer Portal，選取 Certificates。

![alt text](image-2.png)

在類型選擇中，我們要選擇 iOS Distribution (App Store and Ad Hoc)，這就是我們上架 App Store 所需的憑證類型，然後點擊 Continue。

![alt text](image-5.png)

上傳我們剛剛在本地產出的那份 .certSigningRequest (CSR) 檔案。

![alt text](image-6.png)

上傳成功後，蘋果就會立即為我們簽發一張分發憑證。點擊「Download」將它下載到你的 Mac 上。

![alt text](image-7.png)

下載後安裝該憑證，就會看到在 keychain 裡面我們多了一張分發憑證。

接著我們需要匯出這份憑證。在該分發憑證上按右鍵，選取「Export...」。將其儲存為 .p12 格式。系統會提示你設定一個匯出密碼，記住這個密碼，因為稍後在 Pipeline 中我們需要用它來解鎖這個檔案。

![alt text](image-9.png)

### 建立 Provisioning Profile

有了分發憑證後就可以產出描述檔了！同樣回到 Developer，選擇 Profile：

![alt text](image-8.png)

選擇 Distribution 的 App Store Connect。

![alt text](image-10.png)

而因為我們已經在 App Stote Connect 註冊我們的 App 了，所以這裡的下拉選單會出現它，並選擇它

![alt text](image-11.png)

選取我們剛剛建立的分發憑證

![alt text](image-12.png)

自訂一個描述檔名稱，產出後就可以下載了！

![alt text](image-13.png)

## 匯入 Azure DevOps Pipeline

有了分發憑證與描述檔後，我們回到 Azure DevOps，在側邊欄選擇 Pipelines > Library > Secure files

點擊「+Secure file」，分別將剛剛準備好的 .p12 和 .mobileprovision 檔案上傳。Secure Files 是一個加密的儲存空間，專為存放這類敏感資料而設計，確保它們不會以明文形式暴露出來。

![alt text](image-14.png)

![alt text](image-15.png)

然後再到 Variable group 分頁

![alt text](image-16.png)

我們新增一個 Variable group，可以自由命名。在這裡，我們可以建立變數，讓 Pipeline 在執行時可以讀取。

點擊「+Add」，新增一個變數，在 Value 欄位中，輸入你匯出 .p12 檔案時設定的密碼。輸入完畢後，點選右側的「鎖頭」圖示。這會將該變數標記為「Secret」，它的值將會被加密儲存，不會顯示在 UI 上，也無法被複製，更重要的是，它不會在 Pipeline 的 log 中被 print 出來。

![alt text](image-17.png)

## 建立 Pipeline

前置作業準備完畢，接著可以來撰寫我們的 pipeline 了！

### 觸發條件


