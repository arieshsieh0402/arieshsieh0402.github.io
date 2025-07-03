---
title: "Kong API Gateway 真實 client IP 識別"
date: 2025-07-03T10:54:54+08:00
draft: false
categories: ["API Gateway"]
tags: ["Kong","API"]
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

小弟最近要接手管理 APIM 了，會把踩的點記錄下來。

---

在管理 Kong API Gateway 的過程中，發現在啟用 IP Restriction Plugin 並設置白名單後，該 consumer 仍然驗證失敗。經確認此情況與 IP 置換有關。

該 consumer request 路徑為：

`Client → Load Balancer → Kong API Gateway → Backend Service`

在這樣的架構中，Kong 接收到的 IP **並不是實際的 Client IP**，而是來自 Load Balancer 的內部 IP。

經確認：

- Kong 接收到的 client_ip 顯示為內部網段 IP（例如：以 .254 結尾）。
- 這些 IP 實際上是 Load Balancer 的 IP。
- 導致 IP Restriction Plugin 誤判來源 IP，合法用戶被錯誤拒絕。

### Nginx 真實 IP 辨識機制

Kong 是建構在 `Nginx` 之上，利用 `Nginx` 的 `ngx_http_realip_module` 模組來從 HTTP Header 中擷取真正的 Client IP，並改寫 `$remote_addr`。

常見的 HTTP Header：

| Header 名稱        | 說明                                               |
|--------------------|----------------------------------------------------|
| `X-Real-IP`        | 包含直接與第一層代理連線的 Client IP               |
| `X-Forwarded-For`  | 包含完整轉發鏈，列出每一層代理經過的 IP（逗號分隔）|

Kong 設定參數說明：

| 參數名稱           | 說明                                                                 |
|--------------------|----------------------------------------------------------------------|
| `trusted_ips`      | 定義哪些 IP 被視為可信任的代理；只有來自這些 IP 的 Header 才會被信任 |
| `real_ip_header`   | 指定從哪個 Header 擷取真實 IP，預設為 `X-Real-IP`                   |
| `real_ip_recursive`| 啟用後會從轉發鏈中遞迴取得最早的（最靠近 Client 的）IP              |

### 解決方式

- 修改 Kong 設定檔

編輯 /etc/kong/kong.conf，加入以下設定：

```ini
# 信任 Load Balancer 的 IP
trusted_ips = YOUR_LOAD_BALANCER_IP

# 從 X-Forwarded-For Header 取得真實 IP
real_ip_header = X-Forwarded-For

# 啟用遞迴搜尋
real_ip_recursive = on
```

- 重新啟動 Kong

```shell
kong reload
```

Kong 重啟後會重新載入這些設定，透過 `ngx_http_realip_module` 將 $remote_addr 改寫為真實的 Client IP。此時 IP Restriction Plugin 將能正確依據實際來源 IP 進行存取控制。
