---
title: "IIS 部署多個 API 筆記"
date: 2025-08-06T13:51:24+08:00
draft: false
categories: [".NET", "IIS"]
tags: [".NET Core", "API"]
description: "以子應用程式與 .NET Core 設定為例"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---
# 概述

本文紀錄如何在已運行一個 API 服務的 IIS 伺服器上，部署第二個（例如測試用）API。

# 方法：在同一個網站下建立子應用程式 (Sub-application)

這是在現有網站下新增 API 最直接的方法，因為此方法不需要更改 DNS 或防火牆規則（否則你要設定新的 domain name，或是在既有 domain 下使用一個未使用的 port），僅透過 URL 路徑來區分不同的 API。例如：

- 正式 API 呼叫路徑：`http://yourdomain.com/endpoint`
- 測試 API 呼叫路徑：`http://yourdomain.com/test-api/endpoint`


## 原理說明：應用程式集區 (Application Pool) 的隔離機制

「應用程式集區」是 IIS 中實現服務隔離的關鍵。它是一個獨立的容器，由 IIS 工作者處理序 (w3wp.exe) 管理。為新的測試 API 建立一個「全新的、獨立的」應用程式集區主要有以下三個好處：

1.  **隔離性 (Isolation)**：最重要的優點。如果測試 API 因錯誤而崩潰，只會影響其自身的集區，正式運行的 API 服務將完全不受影響，確保了主服務的穩定性。

2.  **安全性 (Security)**：可以為不同的集區設定不同的執行身分 (Identity)，從而對檔案系統、資料庫等資源進行權限隔離。

3.  **獨立設定 (Configuration)**：每個集區可以有獨立的設定，如 .NET 版本、記憶體上限、CPU 使用率限制等，讓正式與測試環境的配置完全分開。


# IIS 設定步驟

1.  **開啟 IIS 管理器**，在左側窗格中找到您現有的網站。
2.  在網站上按右鍵，選擇「**新增應用程式**」。
3.  在「新增應用程式」視窗中，填寫以下資訊：
    - **別名 (Alias)**：設定 URL 路徑，例如 `test-api`。這將成為新 API 的存取路徑的一部分。
    - **實體路徑 (Physical path)**：指向您測試 API 發布檔案所在的資料夾。
    - **應用程式集區 (Application Pool)**：
        - 點擊「**選取...**」。
        - 在彈出視窗中，點擊「**新增...**」來建立一個新集區。
        - 為新集區命名（例如 `TestApiAppPool`）。
        - **(.NET Core 的設定於後面詳述)**
        - 確認選取了這個剛剛建立的新集區。
4.  點擊「**確定**」完成設定。


## 部署 .NET Core API 的特別注意事項

部署 .NET Core / .NET 5+ 的應用程式時，其運作模式與傳統 ASP.NET Framework 不同，設定上有一處關鍵差異。

- **運作原理**：.NET Core 應用程式內建了 Kestrel 伺服器來處理請求。IIS 在此模式下扮演「**反向代理 (Reverse Proxy)**」的角色，它接收外部請求，然後轉發給背景運行的 Kestrel 處理，本身不直接執行 .NET 程式碼。

- **關鍵設定**：基於上述原理，在為 .NET Core API 建立新的應用程式集區時，必須將：
    - **.NET CLR 版本 (.NET CLR version)** 選項設定為 **「沒有受控碼 (No Managed Code)」**。

- **原因**：這樣設定是為了告訴 IIS：「你不需要為這個集區載入 .NET 執行環境，你只需要把流量轉發出去就好」。這不僅能提升效能，也能避免不必要的版本衝突。


# Review 流程

1. **安裝 Hosting Bundle**：確保目標伺服器已安裝對應 .NET 版本的「ASP.NET Core Hosting Bundle」。
2. **發布專案**：將您的 .NET Core API 專案發布到一個獨立資料夾。
3. **新增應用程式**：在 IIS 中按照上述步驟，於現有網站下新增子應用程式。
4. **建立新應用程式集區**：為其建立一個全新的、獨立的應用程式集區。
5. **設定為「沒有受控碼」**：將新集區的 .NET CLR 版本設定為「沒有受控碼」。
6. **測試存取**：透過 `http://yourdomain.com/[Alias]/...` 的路徑來測試新 API。
