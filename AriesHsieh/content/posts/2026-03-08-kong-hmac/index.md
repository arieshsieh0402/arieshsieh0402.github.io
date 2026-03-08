---
title: "Kong HMAC Auth Plugin：筆記與適用場景分析"
date: 2026-03-08T06:57:06+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong", "HMAC", "Authentication", "API"]
description: "整理 Kong HMAC Auth plugin 的運作觀念，以及 HMAC 為何能作為身份驗證之用，並分析其適用與不適用的場景。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

最近工作上在研究 API 認證方案，整理了一個 [Kong HMAC Demo](https://github.com/arieshsieh0402/kong-hmac-demo) 來驗證 HMAC-SHA256 作為 Kong API Gateway 認證機制的可行性。趁還沒忘掉，記錄一下核心觀念與適用場景。

# 什麼是 HMAC

HMAC（Hash-based Message Authentication Code）是一種利用雜湊函數（如 SHA-256）與共享金鑰（shared secret）產生訊息鑑別碼（MAC）的機制。

實際在 HTTP 請求中，client 會選取特定 header（例如 `Date`），以金鑰對其進行 HMAC 運算，產生 signature，再附在 `Authorization` header 中送出。

```
Authorization: hmac username="demo-user",
               algorithm="hmac-sha256",
               headers="date",
               signature="<base64-encoded-signature>"
```

# 為什麼 HMAC 可以作為身份驗證

HMAC 能作為身份驗證的核心原因在於：**只有持有正確金鑰的人，才能產生出合法的 signature。**

驗證流程如下：

1. **Client 計算簽章：** Client 使用共享金鑰對指定 headers 計算 HMAC 簽章，附在請求中送出。

2. **Server 重算比對：** Server（這裡是 Kong）收到請求後，根據 `username` 找到對應的 consumer 及其 secret，對相同的 headers 重新計算一次 HMAC。

3. **比對結果：** 若兩端計算出的 signature 一致，代表請求方持有正確金鑰，身份即可確認。

4. **防重放攻擊：** Kong 的 HMAC plugin 要求簽章必須包含 `Date` header，並設定允許的時間偏移（clock skew）。若請求的時間戳記超出允許範圍（預設 ±300 秒），Kong 會直接拒絕請求，防止攻擊者截取請求後重放。

> HMAC 達成的是 **「你知道這把鑰匙」** 的驗證，本質上是一種基於「共享秘密（shared secret）」的對稱式認證。與非對稱式的 JWT / OIDC 不同，HMAC 不涉及公私鑰對或 Token 發行，設定相對簡單。

# Kong HMAC Plugin 運作流程

以 [kong-hmac-demo](https://github.com/arieshsieh0402/kong-hmac-demo) 為例，整個請求流程如下：

1. Client 取得目前 UTC 時間，格式化為 HTTP-date（例如：`Sun, 08 Mar 2026 07:00:00 GMT`）
2. Client 以 shared secret 對 `date: <HTTP-date>` 字串計算 HMAC-SHA256，並 base64 編碼
3. Client 將 `Date` header 與 `Authorization` header 附在請求中送出
4. Kong 收到請求，從 `Authorization` header 解析出 `username`，找到對應 consumer 的 secret
5. Kong 以相同方式重算 HMAC，比對 signature
6. 比對成功且時間戳在允許範圍內，Kong 將請求代理至 upstream backend

對應的 kong.yml 設定片段：

```yaml
plugins:
  - name: hmac-auth
    config:
      algorithms:
        - hmac-sha256
      clock_skew: 300
      headers:
        - date
```

# 適用場景與限制

## HMAC 適合的場景

HMAC 認證適合 **系統對系統（S2S / machine-to-machine）** 的場景：

- 固定的內部服務互打 API
- 第三方合作廠商串接（每個廠商拿到一組 `username` + `secret`）
- 不需要識別個別使用者、只需確認「是哪個系統發出請求」的情境

## HMAC 的限制：B2B 場景下的個別使用者權限控管

若情境是 **B2B**，且需要針對 end user 做個別權限控管（例如：A 公司的 user1 只能存取部分資源，user2 有不同的權限），HMAC 認證就力不從心了。

原因在於：

| 面向 | HMAC | 細粒度使用者授權需求 |
|------|------|------|
| 識別單位 | Consumer（系統層級） | 個別 end user |
| 授權粒度 | 全有或全無 | 可針對每個使用者設定不同權限 |
| Token 攜帶資訊 | 無（僅簽章） | 可在 token 中夾帶角色、群組、scope 等 claims |
| 動態撤銷 | 困難（需更換 secret） | 可透過 IdP 直接撤銷 Token |

遇到這類需求，有幾個方向可以考慮：

1. **改用 OIDC 認證：** 透過 Kong OIDC plugin，讓 client 攜帶由 IdP（例如 ADFS、Keycloak）發出的 JWT Token。Token 中可夾帶使用者的角色、群組等 claims，Kong 或 upstream 可依此做細粒度授權。

2. **將授權邏輯交給 upstream：** 若 Kong 只負責初步認證（確認請求合法），更細緻的 end user 授權邏輯可以下放給 upstream service 自行處理，Kong 不介入授權判斷。

> 簡單來說，HMAC 解決的是「**你是誰**（哪個系統）」，但無法解決「**你可以做什麼**（個別使用者的授權）」。若兩者都需要，OIDC 或 upstream 自行授權會是更合適的方案。

# 小結

HMAC 認證的優點是輕量、設定簡單，適合 machine-to-machine 的場景。但因為它以 consumer 為單位，缺乏 Token 攜帶細粒度資訊的能力，在需要個別 end user 授權的 B2B 情境下就顯得不足。選擇認證方式前，先想清楚「要識別的是系統還是使用者」，才能選到合適的方案。
