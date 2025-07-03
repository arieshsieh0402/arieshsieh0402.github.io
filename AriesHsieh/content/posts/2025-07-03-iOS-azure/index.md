---
title: "Azure Pipeline 編譯 iOS App Groups 權限未簽入問題"
date: 2025-07-03T16:27:33+08:00
draft: true
categories: ["DevOps", "iOS"]
tags: ["Azure", "Pipeline", "Xcode", "CI/CD", "Signing", "Entitlements"]
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

前言
公司原本的上線流程最近要走 Azure DevOps 方式上線，我維護的 iOS App 不外乎也要走這個流程。原本測試都沒有太大問題，但上線之後發現 App Groups 功能沒有正確運作，就開始查找問題，最終找到問題所在並解決，記錄一下過程，並延伸相關知識。

## Trouble Shooting

一開始覺得很奇怪，這次上線的內容並沒有異動到 App Group 的設定，所以應該跟 source code 沒有太大關係。後來先嘗試在本機 build 一版，結果發現透過這個方式運作正常，但透過 Azure pipeline build 出來的就會發生問題。因此轉向研究 pipeline yaml 的設定是否出現狀況。

先進一步確認透過 Azure pipeline 包出來的 artifacts 內容是否正確，一般而言，iOS App 打包後的成品如下：

```
DistributionSummary.plist
ExportOptions.plist
YourAppName.ipa
Packaging.log
```

### 查看 .ipa

我們可以查看 YourAppName.ipa 裡面到底簽入了哪些權限：

解壓縮 .ipa 檔案：
```sh
unzip YourAppName.ipa -d AppContents
```

進入 Payload 目錄：
```sh
cd AppContents/Payload
```

執行 codesign 指令：
```sh
codesign -d --entitlements :- YourAppName.app
```

每個部分的含義是：
- `codesign`：呼叫程式碼簽署工具
- `d`：表示 "display"，用於顯示簽署資訊
- `-entitlements :-`：要求工具顯示應用程式中包含的所有權限（entitlements）
- 這裡的 `:-` 是一個特殊的語法，表示將輸出導向到標準輸出（stdout）
- `YourAppName.app`：要檢查的應用程式套件路徑

如果有正確簽入 App Groups 的權限，應該要有 `<key>com.apple.security.application-groups</key>` 的蹤跡，但發現透過 Azure pipeline 包出來 .ipa 沒有。

這個時候就進一步查看，是不是在打包開始前根本沒有將權限設定進去。

## DistributionSummary.plist

當你將 iOS App 打包成 .ipa 檔案時，系統會自動生成這個 plist 檔案。它就像是一份「出廠證明書」，記錄了這個應用程式在打包時的所有重要設定和權限資訊包含：
- 應用程式基本資訊
- 權限設定(Entitlements)：如推播、App Groups、iCloud 等
- 簽署資訊：如描述檔(provision profile)、分發憑證(certificate)

而跟今天 App Group 有關的部分，則要查找 Entitlements 的內容。

正確來說應該要長得像下面這樣：

```xml
<key>entitlements</key>
    <dict>
        <key>application-identifier</key>
        <string>com.xxx.xxx</string>
        <key>com.apple.developer.team-identifier</key>
        <string>xxxxxxxx</string>
        <key>aps-environment</key>
        <string>production</string>
        <key>com.apple.security.application-groups</key>
        <array>
         <string>group.yyyyy.yyyyyy</string>
        </array>
        <key>get-task-allow</key>
        <false/>
    </dict>
```

應該要有 `<key>com.apple.security.application-groups</key>` 這個 key，且裡面的值也要正確。

這內容會在 Xcode 設定 App Groups 後，表示新增一個 Entitlement；而每個 Entitlement 最終會被編譯成 .entitlements 檔案，這個檔案會在建置過程中被簽入應用程式中。

因此在 DistributionSummary.plist 中看到 `<key>entitlements</key>` 時，那就是記錄了所有已經被簽入應用程式的權限。

回到問題，我查看透過 Azure pipeline 包出來 DistributionSummary.plist 裡面，確實沒有 App Groups 的蹤跡，這有幾下幾種可能：

### 第一種可能：設定過程的問題
- 專案中可能沒有正確啟用 App Groups 功能
- 在 Xcode 的 Signing & Capabilities 中沒有添加 App Groups
- 或者在添加 App Groups 後，沒有實際設定 App Groups 的識別碼

### 第二種可能：簽署過程的問題
- 使用的開發者帳號可能沒有 App Groups 的權限
- 描述檔（Provisioning Profile）中可能沒有包含 App Groups 的權限
- 簽署過程中可能出現了問題，導致這個權限沒有被正確包含

因為我很確定有正確啟用，所以應該可以排除第一種可能。而第二種可能，也很確定開發者帳號有權限、描述檔也有包含 App Groups，因此問題範圍就縮小到是簽署過程，也就是 pipeline yaml 的設定問題了。

## Xcode@5 task pipeline 設定

Xcode@5 是 Azure Pipelines 提供的一個任務（task），專門用來在雲端執行 Xcode 相關的操作。這個「5」代表的是任務的版本號。

這個任務可以執行多種 Xcode 相關的操作，例如：
- 建置（build）您的 iOS 應用程式
- 封存（archive）應用程式以準備發布
- 執行測試（testing）
- 產生 .ipa 檔案
- 管理程式碼簽署（code signing）

簡單範例如下：

```yaml
- task: Xcode@5
  inputs:
    actions: 'archive'                    # 要執行的動作：建置、封存或測試
    configuration: 'Release'              # 建置組態：Release 或 Debug
    scheme: 'MyApp'                       # Xcode 專案的架構名稱
    xcWorkspacePath: 'MyApp.xcworkspace'  # 工作區檔案的路徑
    packageApp: true                      # 是否打包成 .ipa 檔案
    exportOptions: 'plist'                # 匯出選項的格式
    exportOptionsPlist: 'exportOptions.plist'  # 匯出設定檔的路徑
    signingOption: 'default'              # 程式碼簽署方式
```

這邊有幾個參數值得注意：

### exportOptionsPlist
exportOptionsPlist 是一個為 iOS App 打包過程提供詳細指示的檔案。這個 plist 檔案包含許多重要資訊，如基本發布設定(Team ID，發佈方式是 App Store 還是 Enterprise)、簽署相關設定(分發憑證、描述檔)等。

### signingOption
signingOption 決定了如何處理程式碼簽署，依據微軟官方文件，有以下幾種參數設定：

#### 1. default（專案預設）
這個選項告訴 Xcode「請使用專案中已經設定好的所有簽署配置」。當您使用這個選項時，建置系統會：
- 完全遵循專案的原始簽署設定
- 採用 exportOptionsPlist 中定義的簽署參數
- 不會嘗試自動修改任何簽署相關的設定

#### 2. auto（自動簽署）
選擇這個選項時，告訴 Xcode「請自動處理所有的簽署相關事務」：
- 自動選擇適當的簽署憑證
- 自動管理佈建描述檔
- 可能會忽略一些特定的簽署設定

#### 3. manual（手動簽署）
這個選項給予最大的控制權，告訴系統「我要明確指定所有簽署相關的設定」。使用這個選項時：
- 需要明確提供簽署憑證
- 需要指定具體的佈建描述檔

#### 4. nosign（不簽署）
這個選項告訴系統「完全不要進行程式碼簽署」。

## 解決方式

>**要將此 key 設為 default**

因為在 yaml 檔中，我們同時自定義 exportOptions.plist，手動指定了簽署方式，但又將 signingOption 設為 auto，這就可能導致 Xcode 忽略或覆寫 exportOptionsPlist 中的某些設定，應該也就是造成 App Groups 權限沒有被正確簽入的原因。後來將此 key 設為 default，就成功將 App Groups 權限簽入了～
