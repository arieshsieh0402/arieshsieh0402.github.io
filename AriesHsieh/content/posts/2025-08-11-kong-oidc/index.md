---
title: "Kong OIDC Plugin：Token 驗證筆記"
date: 2025-08-11T09:03:11+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong", "OIDC", "API", "Authentication"]
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

當你的 API 以 OpenID Connect 協定實作 authentication(認證)時，就會使用到 Kong OIDC(OpenID Connect) plugin。本篇要關注的是，當利用 OpenID Connect 協定，讓 Kong 在認證階段自動與 IdP（例如 ADFS）整合，獲取並驗證 JWT Token，保證只有合法使用者能夠存取上游服務的這個過程。

# 驗證流程

當 Kong 收到來自 consumer 的請求後，會進行以下流程：

1. **Issuer Discovery**

Kong 透過 OIDC plugin 中的 issuer 參數，從 .well-known/openid-configuration URL，自動抓取 IdP 的 Metadata。Metadata 包含 `jwks_uri`（JSON Web Key Set）endpoint，用於後續公鑰獲取。

>若在 plugin 的 `extra_jwks_uris` 有設定放置公鑰的 URL，Kong 會先依序對每個 extra_jwks_uris URL 嘗試下載 JWKS。若所有 extra_jwks_uris 都無法成功（連線失敗或無對應公鑰），再從 .well-known/openid-configuration 中的 jwks_uri 嘗試；若未設定 extra_jwks_uris，則會直接使用 .well-known Metadata 提供的 jwks_uri。

2. **Token 取得:** Kong 會對 ADFS 發起請求並獲取並 JWT Token。

3. **公鑰獲取:** Kong 會從剛剛 well-known 或是 extra_jwks_uris 的位置請求公鑰。

4. **簽章驗證:** 解析 JWT `Header.Payload.Signature`。根據 Header 的 `kid`（Key ID）從已下載的 JWKS 中取得對應的公鑰。
使用公鑰執行 RSA256 驗證 `Header` 與 `Payload`。驗證成功表示 token 的內容未被竄改（完整性），token 確實由擁有私鑰的 IdP 簽署（真偽）。

5. **Claim 驗證:** 最後會檢查標準 Claims（exp、iss、aud 等），確保 Token 未過期、來自指定發行者、且受眾正確。

# 小結

以上便是 Kong OIDC plugin 驗證 IdP 所發出的 token 流程，趁還沒有忘掉趕快記下來...XD
