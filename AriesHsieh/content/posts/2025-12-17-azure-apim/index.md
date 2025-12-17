---
title: "Azure Apim 初探-從 Kong 角度來理解"
date: 2025-12-17T15:49:21+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong", "OIDC", "API", "Authentication", "APIM", "Azure"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

最近因為有需求，需要了解一下 Azure APIM 的用法，初步整理一下，從 Kong 的角度來理解新東西。

# 設定 API

## 從 Kong 的觀念來理解 Azure APIM 對應之概念

| Kong | Azure | 差異 |
|--|--|--|
| Service | API (或 Backend) | 在 Kong 中我們使用 Service 來定義上游 URL；Azure 中則在 API 層定義，或使用 Backend 實體來管理共用的上游設定。 |
| Route | Operation | Kong 的 Route 負責路徑匹配；Azure 則在 API 層的 Operation 定義具體 HTTP method 與路徑。 |
| Plugin | Policy | Kong 使用內建 Plugin，直接套用，或使用 lua 寫自訂 plugin；Azure 則使用 XML 定義 Policy。 |
| Consumer | Subscription | Kong 透過 Consumer 身分來控制 API 存取權；Azure 透過 Subscription (訂閱帳戶) 來控制。每一個「訂用帳戶」都會產生一組 **Subscription Key** (Primary & Secondary Key)，呼叫 API 時，於 Header 設置 `Ocp-Apim-Subscription-Key` 來呼叫。|


### Product

在 Azure APIM 中，API 必須被包在一個 Product 中才能被「訂閱」與呼叫(除非該 API 是 open 的，所有人皆可呼叫)。

```
Product(包含多個 API) -> 使用者訂閱 Product -> 取得 Subscription Key -> 呼叫 API
```

>可以把 Azure 的 Product 想像成一個 "Consumer Group"。

## API 之呼叫

### URL 結構

```
https://<Gateway URL>/<API URL Suffix>/<Operation Path>
```

- **Gateway  URL**: 例如 `https://my-apim.azure-api.net`。
- **API URL Suffix**: 建立 API 時設定的後綴 (例如 `v1/demo`)。如果沒設就是空的。
- **Operation Path**: 設定的 Operation 路徑。

**Header (金鑰)**：
Azure APIM 預設會開啟安全性檢查，必須在 Header 帶上訂閱金鑰 (Subscription Key)。
- **Header 名稱**: `Ocp-Apim-Subscription-Key`
- **Header 值**: 訂閱金鑰。
    - 在 Azure Portal 左側選單點選 **Subscriptions**，找到包含該 API 的 Product，點選右側的 `...`，`Show/Hide keys` 來複製 Primary Key。

### OpenID Connect

若有 ROPC 的需求，**Azure APIM**，因為沒有內建「Password Grant」的自動化模組，所以必須手動寫 Policy 來模擬 Kong 的行為。
