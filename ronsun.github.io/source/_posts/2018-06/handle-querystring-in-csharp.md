---
title: 用 C# 處理 query string
date: 2018-06-08 14:40:31
categories:
- C#
- .NET
tags:
---

在網頁開發上, 處理 query stirng 是非常常見的情境, 要手動拆組 query string 也不難, 或是使用 .NET 本身就提供的相關功能讓這件事更輕鬆, 而兩種做法各有優缺點.

<!--more-->

### 手動拆組
最直覺的方法就是手動拆組字串, 例如:

``` csharp
static void HandleQueryString()
{
    // 手動拆 query string 
    var queryString = "?q1=v1&q2=v2&q3=v3";

    if (queryString.StartsWith("?"))
    {
        queryString = queryString.Remove(0, 1);
    }

    var splitedQueryString = queryString
                             .Split('&')
                             .ToDictionary(r => r.Split('=')[0], v => v.Split('=')[1]);

    // 手動組 query string
    var queryDic = new Dictionary<string, string>()
    {
        ["q1"] = "v1",
        ["q2"] = "v2",
        ["q3"] = "v3"
    };

    var formatedQueryString = queryDic
                              .Select(r => $"{r.Key}={r.Value}")
                              .Aggregate((left, right) => $"{left}&{right}");
}
```

手動拆組 query string 常常還要考慮拿到的 query string 前面有沒有多帶問號, 常用的話還要另外抽出來當共用方法, 比較麻煩但相對的很直覺, 且比較不會遇到編碼問題.

### HttpUtility.ParseQueryString
如果不想手動拆組字串的話, 可以用 `HttpUtility.ParseQueryString()` 來處理 query string, 例如: 

``` csharp

static void HandleQueryStringEasier()
{
    // 拆 query string 
    var queryString = "?q1=v1&q2=v2&q3=v3";
    var splitedQueryString = HttpUtility.ParseQueryString(queryString);

    // 組 query string
    var query = HttpUtility.ParseQueryString(string.Empty);
    query["q1"] = "v1";
    query["q2"] = "v2";
    query["q3"] = "v3";
    
    var formatedQueryString = query.ToString();
}
```

這樣做輕鬆很多, 但如果 query string 裡面有中文字或特殊符號的時候容易衍生出編碼相關的問題, 類似的編碼轉換問題在 {% post_link HhttpRequest.QueryString-auto-urlDecode %} 這篇也有提到.

### 結論

兩種做法都各有優缺, 實務上怎麼做還是得視情況決定, 一般來說我還是會優先選擇 `HttpUtility.ParseQueryString`, 如果有無法解決的編碼問題再考慮手動拆組.
