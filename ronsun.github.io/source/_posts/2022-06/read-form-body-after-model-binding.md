---
title: 在 Model Binding 後讀取 Request.Body 的內容
date: 2022-06-26 23:26:11
categories:
- C#
- .NET
tags:
---

一般來說在 WebAPI 的專案中, 我們會偏好使用 Model Binding 的機制來綁定請求內容, 但是之前在一個特殊需求上卻遇到在 Model Binding 後仍然要讀取 `Request.Body` (型別是 `Stream`) 的內容.  

而問題在於, Form Post 的情境中, 在Model Binding 後 (在 Action 中) 讀取`Request.Body` 時會得到空的內容, 但是資料長度又是正確的 (而神奇的是 JSON 和 XML 情境是沒問題的).  

> 這邊 Form Post, JSON 和 XML 情境是依 Content-Type 這個 Header 來分辨的.  

<!--more-->
  
> 這個需求是剛需, 因為這個 API 提供給外部廠商呼叫, 是不固定格式內容的, 所以 Body 中有可能是 Form, JSON ,XML, 甚至是自定義格式的, 比起客製 Model Binding 機制, 簡單的讀取 `Request.Body` 後再解析是比較容易上手的.  

### 解決方式
這個症狀和 Sream 讀取後需要"倒帶" (rewind) 才能再讀取的特性很像, 但是在 Action 中又無法直接倒帶, 剛好之前有遇過類似的問題, 印象中是要額外設定才能開啟倒帶功能 (但我金魚腦忘了怎麼做).  

找了很久沒看到完全一樣的問題, 倒是看到一篇 [類似的情境](https://stackoverflow.com/q/66817215) 值得一試, 只需要簡單的在 `Startup.cs` 中如下啟用就好: 

``` csharp
public void Configure(IApplicationBuilder app)
{
    // Must before UseEndpoints() called.
    app.Use(next => context =>
    {
        context.Request.EnableBuffering();
        return next(context);
    });

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

但有限制: 
+ 一定要出現在 `UseEndpoints()` 被呼叫之前, 推測是因為 Middleware 的執行順序的緣故.  
+ 不能在 Action 中才呼叫 `EnableBuffering()`, 也是合理, 畢竟都在所有 Middleware 後才呼叫, 就不符合上一個限制了, 但可惜目前還不知道是怎樣的運作機制讓倒帶這麼挑場合設定.  

### 結論
基本上能用 Model Binding 就盡量用, 沒特殊情境不需要特別去讀 Body Stream, 解析麻煩而且相對容易錯.  

這要是第一次遇到連關鍵字都沒有不知道會卡多久, 還好之前有先遇過 Stream 倒帶和 Reqeut Rewind 的情境, 真是謝天謝地.  

### 參考
[IAsyncActionFilter, trying to log request body, but it is empty](https://stackoverflow.com/q/66817215)