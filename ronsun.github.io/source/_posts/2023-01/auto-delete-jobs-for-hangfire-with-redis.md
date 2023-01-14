---
title: Hangfire 搭配 Redis 時的資料清理機制
date: 2023-01-14 22:33:13
categories:
- Tools
tags:
---

Hangfire 是很常用的定時排程工具，他會將排程和任務資訊紀錄在資料庫中，但是如果使用 Redis 做資料儲存時，就必須謹慎思考資料清理的問題，畢竟資料存放在記憶體中是不可能不去限制大小的。

<!--more-->

### 限制資料筆數
使用 [Hangfire.Redis.StackExchange](https://github.com/marcoCasamento/Hangfire.Redis.StackExchange) 的話，可以透過 `RedisStorageOptions` 中的 `SucceededListSize` 和 `DeletedListSize` 兩個欄位來限制成功和已刪除的任務的列表長度，避免無限制的增長。  

但是問題是在於失敗的任務怎麼辦?

一開始找了很多網路文章，通常都是建議用 [Job Filters](https://docs.hangfire.io/en/latest/extensibility/using-job-filters.html) 來設定，雖然看起來很合理但實測卻發現失敗列表不可以手動刪除，最後才發現可以透過 `AutomaticRetryAttribute` 來達到目標。  

這個做法的主要邏輯是在重試次數結束後刪除，使得任務進入已刪除列表，而已刪除列表是可以設定最大長度的， `AutomaticRetryAttribute` 可以單獨用在需要的排程上，也可以一次設定到所有排程，如下:  
``` csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services
        .AddHangfire((serviceProvider, config) =>
        {
            config
                .UseFilter(new AutomaticRetryAttribute { Attempts = 0, OnAttemptsExceeded = AttemptsExceededAction.Delete })
                ;
        })
}
```

### 結論
失敗列表不能手動刪除的這個限制滿合理的，畢竟有重試機制存在，如果可以手動刪除那等於是和重試機制產生衝突了，倒是刪除失敗任務的設定在重試機制的設定中這點滿讓人意外的。  

然後網路的解法不能盡信，要實驗過才知道能不能運作。