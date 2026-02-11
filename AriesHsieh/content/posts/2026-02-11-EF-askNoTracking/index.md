---
title: "Entity Framework AsNoTracking 效能優化筆記"
date: 2026-02-11T08:48:27+08:00
draft: false
categories: ["Database"]
tags: ["Entity Framework", "Performance", ".NET", "SQL", "Optimization"]
description: "記錄 Entity Framework 只讀查詢中使用 AsNoTracking 的重要性"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

在進行 Entity Framework 查詢時，當你執行的是「唯讀」操作，卻沒有使用 `AsNoTracking()` 時，效能會有差距。本篇記錄實際踩雷的經驗——當資料量小時還看不出來，但隨著資料成長，超過預設的 60 秒 Timeout 限制，導致排程工作失敗。

# 問題情景

某個 .NET Framework 排程工作，定期執行一份複雜的資料查詢：

```csharp
using (var db = new MyDbContext())
{
    var result = db.Orders
        .Where(o => o.Status == "Completed")
        .GroupBy(o => o.CustomerId)
        .Select(g => new
        {
            CustomerId = g.Key,
            TotalAmount = g.Sum(o => o.Amount),
            OrderCount = g.Count()
        })
        .ToList();

    // 處理 result...
}
```

起初，資料量只有幾千筆，執行速度很快。但隨著程式修改，時間推移，累積到數十萬筆以上，這個查詢開始變得異常緩慢，最終超過預設的 60 秒 Timeout，排程執行失敗。

# 根本原因：Change Tracking

Entity Framework 預設會追蹤查詢結果中的每一個實體：

1. **記憶體使用**：EF 會在內部維護一個追蹤 dict，記錄每個實體的狀態（Unchanged、Modified 等）
2. **快照儲存**：EF 會保存實體的原始狀態，以便在 SaveChanges 時偵測變更
3. **效能損耗**：當結果集很大時，這些額外操作會顯著影響查詢執行時間

對於只讀查詢，這些追蹤機制是沒有必要的浪費。

# 解決方案：AsNoTracking()

在查詢上加上 `AsNoTracking()`，告訴 EF 你不需要追蹤這些實體：

```csharp
using (var db = new MyDbContext())
{
    var result = db.Orders
        .AsNoTracking()  // 停用追蹤
        .Where(o => o.Status == "Completed")
        .GroupBy(o => o.CustomerId)
        .Select(g => new
        {
            CustomerId = g.Key,
            TotalAmount = g.Sum(o => o.Amount),
            OrderCount = g.Count()
        })
        .ToList();

    // 處理 result...
}
```

加上 `AsNoTracking()` 後，相同的查詢執行時間從超過 60 秒下降到數秒，效能提升顯著。

趁著這次踩雷還記得清楚，趕快記下來...XD
