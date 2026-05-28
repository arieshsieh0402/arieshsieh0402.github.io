---
title: "Kong AI MCP Proxy 撞上 OpenID Connect"
date: 2026-05-28T21:04:08+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong", "MCP", "OIDC", "AI", "Microsoft Entra ID"]
description: "在 Kong 3.12 上同時掛 ai-mcp-proxy 與 openid-connect plugin 時遇到的相容性問題、推論過程與解法。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

最近在 Kong 3.12 上做一個 PoC，想把公司內部的 RESTful API 透過 `ai-mcp-proxy` plugin 包成 MCP server，讓 Microsoft Copilot Studio 可以呼叫。同時為了讓呼叫者帶身分過來，在同一個 route 上掛了 `openid-connect` plugin，搭配 Microsoft Entra ID（原 Azure AD）做驗證。

理論上很單純：Copilot Studio 帶 token，Kong 驗證，通過後轉成對後端 REST API 的呼叫。實際做下去踩了一些雷，趁還沒忘掉趕快記下來。

# 架構概觀

```
Copilot Studio
    │ (Bearer token from Entra ID)
    ▼
F5 (load balancer)
    │
    ▼
Kong Data Plane (Route /My-MCP)
    │ ┌─ openid-connect plugin (驗證 Entra ID token)
    │ └─ ai-mcp-proxy plugin (mode: conversion-listener)
    ▼
Backend RESTful API (Azure App Service)
```

# 401 Invalid Token

OpenID Connect plugin 一掛上去，設定大致是這樣：

```yaml
- name: openid-connect
  config:
    issuer: https://login.microsoftonline.com/<tenant-id>/v2.0
    issuers_allowed:
      - https://login.microsoftonline.com/<tenant-id>/v2.0
    auth_methods:
      - bearer
      - session
```

Copilot Studio 一打過來就是 `401 invalid token`。看 Kong log 發現：**Entra ID 發的 token，`iss` claim 不一定是 v2.0 格式**。

Entra ID 的 token 有兩種版本：

- **v1.0 token**：`iss` 為 `https://sts.windows.net/<tenant-id>/`
- **v2.0 token**：`iss` 為 `https://login.microsoftonline.com/<tenant-id>/v2.0`

實際發哪一種，跟 App Registration manifest 中的 `accessTokenAcceptedVersion`、以及取 token 時用的 endpoint 都有關。Copilot Studio 那邊我不容易控制，所以先選擇在 Kong 端同時接受兩種 issuer：

```yaml
issuers_allowed:
  - https://login.microsoftonline.com/<tenant-id>/v2.0
  - https://sts.windows.net/<tenant-id>/   # 注意結尾的斜線
```

> 注意 trailing slash：`sts.windows.net/<tenant-id>/` 結尾的斜線必須跟 token 內的 `iss` 完全一致，少一個斜線會比對失敗。

# 500 Internal Server Error，port is nil

issuer 加上後 401 確實消失，但變成 500。Kong error log 顯示：

```
[error] [openid-connect] ./openid-connect/arguments.lua:705:
attempt to concatenate local 'port' (a nil value), client: unix:
```

`port is nil` 在 Lua 中是程式碼想把一個 nil 變數當字串串接時爆掉，這表示：**OpenID Connect plugin 在某段邏輯需要 port，但取到 nil**。

## 卡很久的錯誤推論

一開始我推論問題在 `auth_methods` 同時開了 `bearer` 跟 `session`。理由是：純 bearer 流程不需要 redirect URI，但 session 流程的初始化會需要；當 redirect URI 為 null 時 plugin 會自動推導 host:port[^kong-13147]，**推導失敗就 nil**（這一步是在沒有 plugin 原始碼可看的前提下做的猜測，後面會看到這整個推論方向其實是錯的）。

實測時把 `auth_methods` 改成只有 `bearer`，500 真的消失了 — 看起來推論對了。

但接著我同時把 `issuers_allowed` 改回只有 v2.0 做別的測試，結果又變回 issuer 不匹配的錯誤。當我把 v1.0 issuer 加回去，500 又出現了。整理一下：

| auth_methods | issuers_allowed | 結果 |
|---|---|---|
| `[bearer, session]` | 只有 v2.0 | 401（iss 不匹配） |
| `[bearer, session]` | v1.0 + v2.0 | **500 port nil** |
| `[bearer]` | 只有 v2.0 | required issuers are missing |
| `[bearer]` | v1.0 + v2.0 | **500 port nil** |

發現 **500 跟 `auth_methods` 是否有 session 無關，而是「token 通過 issuer 白名單後」才會觸發**。

之前的推論之所以錯，是因為我同時改了兩個變數。後來純粹只動 `auth_methods` 的對照測試才發現真正的問題。

## `client: unix:`

回頭仔細看 error log，有一個細節之前被忽略了：

```
[error] [openid-connect] ... port is nil ..., client: unix:, server: kong,
request: "GET /My-MCP/my_tool?q=...&language=zh-TW HTTP/1.1"
```

`client: unix:` 代表這個請求不是從 TCP 連線進來的，而是**從 Unix socket 進來的**。

> Unix socket（Unix Domain Socket）跟 TCP socket 都是程式之間收發訊息的機制，差別在對方怎麼找：TCP socket 用 IP + port、走網卡；Unix socket 用檔案系統路徑（例如 `/var/run/nginx.sock`），限於同一台機器內，不走網卡、也沒有 IP/port 概念[^unix-socket]。
>
> Nginx/Kong 在內部要產生子請求時 — 例如 ai-mcp-proxy 把 MCP 轉成 REST 呼叫後再丟回 Kong 跑一次 plugin chain — 會在本機開一個 Unix socket、用 `proxy_pass` 把請求遞回去[^nginx-unix]，不用繞網路一圈。
>
> 這條子請求走到 OIDC plugin 時沒有 source IP、也沒有 source port，Nginx log 就用 `client: unix:` 來標記。

另外也注意到，Copilot Studio 打的是 `POST /My-MCP`（MCP 協定是 POST JSON-RPC），但這裡卻是 `GET /My-MCP/my_tool?q=...`。對照同時間的 ai-mcp-proxy log：

```
[ai-mcp-proxy] received rpc request: tools/call id: 3, client: <client-ip>,
request: "POST /My-MCP", request_id: "d1623a13..."

[error] [openid-connect] arguments.lua:705: port is nil, client: unix:,
request: "GET /My-MCP/my_tool?q=...", request_id: "1509b610..."
```

兩個請求的 `request_id` 不同，代表是**獨立的請求生命週期**。事情就清楚了：

> **ai-mcp-proxy 在 conversion-listener 模式下，收到 MCP `tools/call` 之後，會在 Kong 內部產生一個新的 HTTP 請求，透過 Unix socket 走回 Kong 自己的 plugin chain，目的是把 MCP 協定轉成標準 REST 呼叫送到 upstream。**

而 Unix socket 沒有 port 的概念，當這個內部子請求觸發了 OIDC plugin 中需要 port 的邏輯時，就觸發 `port is nil`。

這也解釋了之前的觀察：

- 為什麼 500 只在 issuer 通過後出現？因為要先讓外部請求（POST tools/call）通過 OIDC 驗證，ai-mcp-proxy 才會收到 MCP 訊息，內部子請求才會被產生。
- 為什麼 `auth_methods` 改成 bearer 仍然 500？因為跟 `auth_methods` 沒關係，是內部子請求的問題。

> 關於 `arguments.lua:705` 確切是哪段邏輯，因為 Kong OpenID Connect 是 Enterprise plugin、原始碼非公開，我無法 100% 確認。但「Unix socket 沒有 port」是 Nginx/OpenResty 層級的事實，跟 Kong GitHub Discussion #13147[^kong-13147] 中提到的「OIDC plugin 會自動推導含 port 的 URL」吻合。

# 拆分 ai-mcp-proxy 的 mode

查明原因後，問題變成：**怎麼讓 ai-mcp-proxy 的內部子請求不被 openid-connect plugin 攔截**。

根據 Kong 官方文件，ai-mcp-proxy 有四種 mode：

| mode | 用途 | 是否產生內部子請求 |
|---|---|---|
| `conversion-listener` | 單一 plugin 完成 MCP 轉換 + 處理 | 是（原本用的） |
| `conversion-only` | 只定義 tool 規格，不處理請求 | 不直接處理 |
| `listener` | 聚合多個 conversion-only 的 tools | 是 |
| `passthrough-listener` | 透傳到已是 MCP server 的後端 | 否（但要求後端是 MCP server） |

`passthrough-listener` 不可行，因為我的後端是 REST API、不是 MCP server。剩下的選項是 **拆成 conversion-only + listener 兩個 route**，讓 OIDC 只掛在 listener 上，讓內部子請求走到沒有 OIDC 的 conversion-only route。

老實說，這個方向當下沒有官方文件保證能繞過 plugin chain，但也是可以試試。

## 拆分後的架構

```
Copilot Studio
    │ (Bearer token)
    ▼
F5 → Kong
    │
    ▼
Route A: /My-MCP             ← 對外，Copilot Studio 打這個
    plugins:
      - ai-mcp-proxy (mode: listener, server.tag: my-mcp-tools)
      - openid-connect          ← OIDC 只掛這裡
    │
    │ (內部子請求，透過 Unix socket)
    ▼
Route B: /My-MCP-Conversion  ← 內部用，不對外
    plugins:
      - ai-mcp-proxy (mode: conversion-only, tags: [my-mcp-tools])
      ❌ 不掛 openid-connect
    service: 後端 REST API
```

- 對外 path 放在 listener route。
- 後端 service 掛在 conversion-only route（因為 tool 規格要對應到實際 API）。
- tag 必須一致（listener 用 `server.tag` 找到對應的 conversion-only）。
- conversion-only route 不掛 OIDC（讓內部子請求不被攔截）。

結果 500 port nil 消失，OIDC 驗證在 listener route 正常運作，內部子請求順利走到 conversion-only route 沒被攔截。

# upstream timeout

但隨之又有新的問題：呼叫 tool 後，後端 timeout。Kong access log 顯示：

```json
{
  "response": { "status": 499, "size": 0 },
  "upstream_status": "-",
  "latencies": { "proxy": 9999, "request": 10000 }
}
```

`9999ms` 這個數字很奇怪，service 的 `read_timeout` 是 60000ms，不該卡在 10 秒。查了一下，發現 ai-mcp-proxy 的 listener 有自己的 `config.server.timeout`，**預設 10 秒**。後端的語意搜尋處理需要超過 10 秒，所以被 plugin 自己 cut 掉了。

調到 30 秒後就通了：

```yaml
- name: ai-mcp-proxy
  config:
    mode: listener
    server:
      tag: my-mcp-tools
      timeout: 30000   # 從預設 10 秒調到 30 秒
```

整條鏈路打通。

# 最終可運作的設定

## Route A：對外的 MCP listener

```yaml
services:
  - name: mcp-listener-service
    url: http://placeholder  # listener 不轉發到 upstream，這個 url 不會被用到
    routes:
      - name: my-mcp-listener
        paths:
          - /My-MCP        # 對外 path，Copilot Studio 設定的 endpoint
        strip_path: true
        plugins:
          - name: ai-mcp-proxy
            config:
              mode: listener
              server:
                tag: my-mcp-tools
                timeout: 30000
          - name: openid-connect
            config:
              issuer: https://login.microsoftonline.com/<tenant-id>/v2.0
              issuers_allowed:
                - https://login.microsoftonline.com/<tenant-id>/v2.0
                - https://sts.windows.net/<tenant-id>/
              auth_methods:
                - bearer
              verify_claims: true
              verify_signature: true
```

## Route B：內部的 conversion-only

```yaml
services:
  - name: my-backend-service
    url: https://<upstream-host>
    routes:
      - name: my-mcp-conversion
        paths:
          - /My-MCP-Conversion   # 內部用，不對外
        strip_path: true
        plugins:
          - name: ai-mcp-proxy
            tags:
              - my-mcp-tools     # 必須跟 listener 的 server.tag 一致
            config:
              mode: conversion-only
              tools:
                - description: Some tool description
                  method: GET
                  path: /My-MCP-Conversion/my_tool
                  annotations:
                    title: my_tool
                  parameters:
                    - name: q
                      in: query
                      required: true
                      schema: { type: string }
                    - name: language
                      in: query
                      schema: { type: string }
                    - name: limit
                      in: query
                      schema: { type: integer }
                    - name: categories
                      in: query
                      schema: { type: string }
                # ... 其他 tools
        # 不掛 openid-connect
```

# 回顧

## 關於 debug 過程

回頭看，有兩件事值得記下來：

**1. 同時改多個變數，把單一觀察當成因果**

把 `auth_methods` 從 `[bearer, session]` 改成 `[bearer]` 時 500 消失了，但同時我也把 `issuers_allowed` 縮減了，等於同時動了兩個變數。把錯誤的「session 造成 500」當結論，後續分析方向就偏了一陣子。回到單一變數測試後，真相才浮現：500 跟 auth_methods 完全無關。

> debug 時改設定一次只改一個。看到「改了 X 之後問題消失」也要警惕是不是同時動了 Y。

**2. 拆分方案的成功率好像很低？**

當時我覺得「ai-mcp-proxy 的內部子請求機制是 plugin 內建的，改 mode 拆分不太可能繞過去」，給了不高的成功機率推論。實際試了之後就成功了。

> 成本低、風險可控的時候，直接做實驗比花時間推論成敗划算。對 Kong 內部行為的推論，本來就是有限資訊下的猜測，實測看看最快。

## 關於 Kong AI MCP Proxy + OIDC 的相容性

整理一下這次的結論：

- **conversion-listener 模式 + openid-connect plugin 同一 route 不相容**（在 Kong 3.12）。
- 原因是 conversion-listener 會產生 Unix socket 內部子請求，觸發 OIDC plugin 中需要 port 的程式碼。
- 拆成 conversion-only（後端 + tool 規格）+ listener（對外 + OIDC）兩個 route 可以繞過。
- 對外 path 維持在 listener route，Copilot Studio 那邊不用改設定。

我不確定這算 plugin bug 還是「不支援的組合」，但 Kong 官方文件中 MCP 場景的範例都是用 `ai-mcp-oauth2`（專為 MCP 設計的 OAuth2 plugin），而不是通用的 `openid-connect`。如果你的場景能接受用 `ai-mcp-oauth2`，可能更符合 Kong 設計意圖。

# 小結

1. **不要**把 ai-mcp-proxy 設成 `conversion-listener` 模式 + 同 route 掛 OIDC，會 500 port nil。
2. 拆成兩個 route：listener（對外、掛 OIDC）+ conversion-only（後端、不掛 OIDC）。
3. 兩個 route 用 tag 串起來（listener 的 `server.tag` = conversion-only 的 plugin tag）。
4. 對外 path 放在 listener route，client 端不用改設定。
5. listener 的 `server.timeout` 預設 10 秒，後端慢的話記得調高。
6. Entra ID 的 token issuer 有 v1.0（`sts.windows.net`）跟 v2.0（`login.microsoftonline.com`）。
7. trailing slash 要跟 token 的 iss 完全一致，差一個斜線就比對失敗。

本文基於 Kong 3.12 + ai-mcp-proxy + openid-connect 的組合，Kong 版本演進可能會修掉這個相容性問題，未來版本不一定需要這樣拆分。

[^unix-socket]: https://man7.org/linux/man-pages/man7/unix.7.html
[^nginx-unix]: https://nginx.org/en/docs/http/ngx_http_upstream_module.html
[^kong-13147]: https://github.com/Kong/kong/discussions/13147
