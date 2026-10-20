---
title: "Kong reload 後設定沒有恢復：從 .kong_env 合併機制理解註解與明確設定"
date: 2026-10-20T14:38:41+08:00
draft: true
categories: ["API Gateway"]
tags: ["Kong", "Data Plane", "Nginx", "Configuration"]
description: "記錄 Kong Data Plane 執行 reload 時，將設定註解掉未恢復預設值的排查過程，以及 .kong_env 合併機制帶來的設定差異。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

最近在調整 Kong Data Plane 的回應標頭時，遇到一個讓我有點困惑的現象：

**同樣修改 `kong.conf`，restart 後符合預期，reload 卻像是沒有吃到設定。**

一開始以為是 reload 沒有成功，後來才發現，問題不是「新設定沒有讀到」，而是「我沒有明確提供新值」。

# 發現問題：註解掉 off，卻沒有恢復

我的環境是安裝在 VM 上的 Kong Data Plane。

當時為了關閉 Kong 回應標頭，在 `kong.conf` 加入：

```ini
headers = off
```

需要恢復時，我的操作很直覺：把這行註解掉。

```ini
# headers = off
```

我原本的理解是：

> 不再指定 `off`，應該就會恢復預設值。

但執行 reload 後，透過 Postman 呼叫 API，仍然沒有看到預期恢復的 `Via` 標頭；改用 restart，結果才符合預期。

# 排查：明確指定值，reload 就生效了

排查過程中，我也嘗試在 Postman 請求加入 `Connection: close`，但沒有改變測試結果。

真正的轉折，是不再透過註解切換，而是明確寫入：

```ini
headers = server_tokens, latency_tokens, X-Kong-Request-Id
```

再次 reload，`Via` 就恢復了。

這讓問題變得更明確：

**reload 可以套用明確指定的新值，但將設定註解掉，並沒有清除先前的值。**

# 原始碼線索：舊設定加上新設定

接著查看 [Kong 官方 `reload.lua` 原始碼](https://github.com/Kong/kong/blob/21032122ab80c973f368f6bc539f53985e7164e8/kong/cmd/reload.lua#L21-L33)，其中有一段合併設定的邏輯：

```lua
local kong_env = assert(pl_file.read(args.conf))

if pl_path.exists(new_config.kong_env) then
  kong_env = assert(pl_file.read(new_config.kong_env)) .. "\n" .. kong_env
end
```

這段程式不是先逐項比較設定，而是把兩份檔案的文字接起來：

```text
既有 .kong_env 的內容
＋ 換行
＋ 這次指定的設定檔內容
```

接著寫回 `.kong_env`，再交給 `conf_loader` 載入。

要注意，這段合併邏輯有條件：**指定了 `-c`，而且指定的設定檔存在。** 不能只看這一段，就把所有 reload 呼叫方式都視為完全相同。

# 為什麼註解沒有清掉舊值？

套到這次的情境，舊 `.kong_env` 已經有：

```ini
headers = off
```

而新設定檔只有：

```ini
# headers = off
```

合併後，相當於：

```ini
headers = off
# headers = off
```

這時就比較容易理解了：

**第二行只是註解，並不是「刪除第一行」的指令，也沒有提供新的設定值。**

因此，與其說「註解蓋不過舊設定」，不如說：

> 舊設定被保留下來，而新內容沒有提供替代值。

嚴格來說，`reload.lua` 直接呈現的是文字合併流程；設定檔如何解析，還會經過 [`conf_loader`](https://github.com/Kong/kong/blob/21032122ab80c973f368f6bc539f53985e7164e8/kong/conf_loader/init.lua#L127-L169)。本次實測顯示，明確指定新值可以成功切換，單純註解沒有達到相同效果。

# 我的操作調整

經過這次測試，我會避免用「註解／取消註解」來切換這個參數，而是保留一行有效設定，明確修改它的值。

關閉時：

```ini
headers = off
```

恢復時：

```ini
headers = server_tokens, latency_tokens, X-Kong-Request-Id
```

再執行：

```bash
kong reload -c /etc/kong/kong.conf
```

這不是要把 reload 判定為不可靠，而是要區分兩件事：

- **明確設定新值**
- **移除原本的設定宣告**

在這次測試中，兩者並不等價。

# 最後的心得：檔案內容與執行狀態，不一定是同一件事

這次最值得記住的，不只是 `headers` 的操作方式，而是：

**看到設定檔裡某行已被註解，不代表執行中的系統就一定恢復預設值。**

如果載入流程會保留舊狀態，再合併新內容，那麼「沒有指定」可能代表沿用，而不是重設。

對設定管理而言，除了檢查檔案，也需要驗證實際執行結果。這次就是透過明確改值、觀察回應，再回頭看原始碼，才釐清兩者的差別。

# 驗證範圍

本文記錄的是本次 VM 環境中的實測結果。引用的 Kong 原始碼為 2026-10-20 查閱 `master` 分支時的 commit `2103212`；不同 Kong 版本或不同 reload 呼叫方式，實作與結果可能不同。
