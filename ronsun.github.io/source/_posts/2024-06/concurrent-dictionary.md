---
title: 在非同步情境下使用 ConcurrentDictionary 
date: 2024-06-22 23:53:14
categories:
- C#
- .NET
tags:
---

在處理非同步操作時，當多個非同步任務同時操作共享資源時，可能會導致資料不一致。這篇文章將聚焦於如何在非同步情境下使用 `ConcurrentDictionary` 取代 `Dictionary` 來解決這些問題。

<!--more-->

### 問題程式碼
``` csharp
[HttpGet]
public async Task<IActionResult> Foo()
{
    var dic = new Dictionary<long, long>();
    var tasks = new List<Task>();
    var rdm = new Random();
    int i = 0;
    while (i++ < 10)
    {
        // Do NOT await here.
        var t = HttpRequest(dic, rdm.Next());
        tasks.Add(t);
    }

    await Task.WhenAll(tasks);

    return Ok();

    async Task HttpRequest(Dictionary<long, long> dic, int num)
    {
        await Task.Delay(500);
        dic[num] = num;
    }
}
```

在這段程式碼中，我們模擬同時發送多個 Http Request 的操作，由於多個請求不需互相等待，因此每次呼叫 HttpRequest 方法時並不會加上 `await` 關鍵字，而是在所有請求都發出後再等待所有結果完成，以達到效能目標。

但每個 Http Request 完成後都會操作一個共享的 `Dictionary<long, long>` 物件，這在非同步情境下引發多執行緒操作時，會有非常小的機率導致資料不一致。

### 解決方案：使用 `ConcurrentDictionary`
為了解決上述問題，我們可以將 `Dictionary` 換成 `ConcurrentDictionary` 這個執行序安全的類別，允許多個執行緒安全地操作其中的資料。  

以下是修改後的程式碼：

``` csharp
[HttpGet]
public async Task<IActionResult> Foo()
{
    var dic = new ConcurrentDictionary<long, long>();
    var tasks = new List<Task>();
    var rdm = new Random();
    int i = 0;
    while (i++ < 10)
    {
        // Do NOT await here.
        var t = HttpRequest(dic, rdm.Next());
        tasks.Add(t);
    }

    await Task.WhenAll(tasks);

    return Ok();

    async Task HttpRequest(ConcurrentDictionary<long, long> dic, int num)
    {
        await Task.Delay(500);
        dic[num] = num;
    }
}
```

### 結論
其實當初開發時有閃過一點疑慮，但因為這只是一個賦值的極簡單操作，一時輕率以為不會出錯，導致在上線後發生資料不一致的問題，且因為這個問題一個月只發生一兩次所以多花了很多時間蒐集資訊和追查原因，真的划不來。除了 `ConcurrentDictionary` 外，從它的命名空間可以發現還有其他針對執行緒安全所設計的類別，也應善加利用。  

### 參考  
ChatGPT