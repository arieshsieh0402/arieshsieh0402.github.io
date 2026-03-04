---
title: "Kong 與 Nginx 之設定流向"
date: 2026-03-04T21:44:38+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong", "Nginx", "SSL", "TLS"]
description: "整理 Kong 設定如何流向至 Nginx 實際生效的過程，以 SSL Ciphers 為例。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# Kong 與 Nginx 設定關係筆記

工作上遇到一些狀況，需要處理 HTTPS 加解密演算法，剛好趁這個機會記錄一下 Kong 跟 Nginx 的關係。

這邊以 SSL Ciphers 為例，整理 Kong 設定如何流向至 Nginx 實際生效的過程。

## 概念架構

Kong 的設定與 Nginx 的設定之間存在著三層結構：

| 層級 | 檔案位置 | 用途 |
|------|---------|------|
| **源頭** | `kong.conf` | Kong 的主設定檔，定義所有運行參數 |
| **入口** | `nginx.conf` | Nginx 的起點，僅做全域/事件設定，在 `http {}` 內 include 真正的設定檔 |
| **生效** | `nginx-kong.conf` | 實際生效的設定，包含 ssl_ciphers、ssl_protocols、各個 listen 等 TLS/Server 設定 |

簡單來說：kong.conf 是來源、nginx.conf 是入口、nginx-kong.conf 是實際生效結果。

## 設定流向 - 以 SSL Ciphers 為例

### Step 1: 源設定
在 `kong.conf` 設定 ssl_cipher_suite 和 ssl_ciphers：

```conf
ssl_cipher_suite = custom
ssl_ciphers = ECDHE-RSA-AES256-GCM-SHA384:...
```

### Step 2: Kong 翻譯
當你執行 `kong start` 或 `kong restart` 時，Kong 會根據 kong.conf 生成 Nginx 的實際設定。

### Step 3: Nginx 生效
真正生效的內容會被寫入 `nginx-kong.conf`，包含翻譯後的 ssl_ciphers、ssl_protocols 等設定。
