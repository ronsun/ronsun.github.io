---
title: WebAPI 安全的取得 Request Body
date: 2018-11-18 00:46:45
categories:
- C#
- .NET
tags:
---

標題下的有點奇怪, 框架預設提供 Model Binding 來將請求內容(request body)綁定到參數中, 其實一般情境不需要手工去讀取它的. 但是之前遇到一個情境是需要在 ActionFilterAttribute 中取得請求內容並紀錄到日誌 (log) 中, 這時候問題就來了.  

<!--more-->

先來段程式碼, 看起來很正常, 就是在 ActionFilterAttribute 中把請求內容抓出來, 然後記錄到日誌中收工, 但接下來有個問題...  

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        using (Stream s = actionContext.Request.Content.ReadAsStreamAsync().Result)
        using (StreamReader sr = new StreamReader(s))
        {
            s.Seek(0, SeekOrigin.Begin);
            var body = sr.ReadToEnd();
            // Logging here...
            logger.Info(body);
        }
    }
}
```

上面說到紀錄完日誌收工, 這時候我們回到 WebAPI 中的 Action 來看看下面這段程式碼, 可以看到 Action 中又再次讀取一次請求內容, 但是這次卻會拋出例外(exception), 這就是有問題的地方了.  

``` csharp
public class ValuesController : ApiController
{
    [HttpPost]
    [DemoActionFilter]
    public void Post()
    {
        string data = string.Empty;
        using (Stream s = Request.Content.ReadAsStreamAsync().Result)
        using (StreamReader sr = new StreamReader(s))
        {
            s.Seek(0, SeekOrigin.Begin);
            data = sr.ReadToEnd();
        }
    }
}
```

> `Stream.Seek(0, SeekOrigin.Begin)` 是用來重置 stream 的讀取位置的, 也可以這樣寫 `Stream.Position = 0`

### 不用 using
簡單暴力的做法就是把 DemoActionFilterAttribute 中讀取時的 `using`拿掉, 這樣資源就不會被強制回收, 如下:  

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        Stream s = actionContext.Request.Content.ReadAsStreamAsync().Result;
        StreamReader sr = new StreamReader(s);
        s.Seek(0, SeekOrigin.Begin);
        string body = sr.ReadToEnd();
        // Logging here...
        logger.Info(body);
    }
}
```

或是初始化 StreamReader 的時候要用參數最多的那個多載, 並把最後一個參數 leaveOpen 設成 true 讓 stream 本身在 StreamReader 釋放後不會被關閉, 但這樣還要特別去注意其他參數預設要填什麼有點麻煩, 而且也不確定 `ReadAsStreamAsync()` 這個方法使會不會建立一個新的 Stream 以及是否必須手動釋放 , 例如下面的範例

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        Stream s = actionContext.Request.Content.ReadAsStreamAsync().Result;
        using (var sr = new StreamReader(s, Encoding.UTF8, true, 1024, true))
        {
            s.Seek(0, SeekOrigin.Begin);
            string body = sr.ReadToEnd();
            // Logging here...
            logger.Info(body);
        }
    }
}
```

這兩種做法雖然是能讓程式碼能成功運作, 但是既然這些類別有實作 IDisposable 來允許使用 using 來提早回收資源, 在不完全確定這些資源能被 GC 即時回收的情況下, 冒然把 using 拆掉似乎不是首選.

### 改用 ReadAsStringAsync
這是最簡短的作法, 且這樣做在 Action 中要再次讀取 Sream 時並不會出錯, 但是卻有個限制 - **Action 不能有參數**.  
``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        string body = actionContext.Request.Content.ReadAsStringAsync().Result;
        // Logging here...
        logger.Info(body);
    }
}
```

從[這張圖](https://www.asp.net/media/4071077/aspnet-web-api-poster.pdf)可以看出, 如果 Action 有參數的話 (像是這樣 `public void Post(MyModel model)` ), 會先做 Model Binding (在圖中標註 C 點的地方) 然後才是做 ActionFilter, 所以當進到 ActionFilterAttribute 裡面時, 請求已經被讀取過了, 這時候讀出來的結果會是空字串.  

因此, 這種做法必須限制 Action 不能有參數, 但是只為了 ActionFilterAttribute 需要而限制的 Action 的實作方式是不合理的, 會造成未來擴展或維護上的困擾.  

### 複製 Stream
既然要讀取內容又不能把資源釋放掉, 不如直接複製一份出來讀取, 再釋放掉複製版的資源就好了.  

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        using (Stream s = new MemoryStream())
        {
            actionContext.Request.Content.CopyToAsync(s);
            // need call it twice if model binding works
            actionContext.Request.Content.CopyToAsync(s);
            using (StreamReader sr = new StreamReader(s))
            {
                s.Seek(0, SeekOrigin.Begin);
                string body = sr.ReadToEnd();
                // Logging here...
                logger.Info(body);
            }
        }
    }
}
```

之前在某一本書上看到複製 stream 不好, 但當時沒有仔細看, 所以也不清楚為什麼不好, 不過倒是有在網路上找到一篇關於[複製 stream 的效能問題](http://vunvulearadu.blogspot.com/2013/04/steamcopyto-performance-problems.html), 基本上是因為短時間大量資料湧入再加上資源來不及釋放, 導致記憶體用量飆高, 這邊的作法跟他提到的解法有點不同, 但原則都是盡早釋放資源.  

另外還有一個問題是, 這個做法如果跟 Model Binding 一起使用, `actionContext.Request.Content.CopyToAsync(s)` 要執行兩次才能才能正確的複製 stream, 推測跟 Stream.Position 有關但不知道怎麼證實.  

### 內建的 Model Binding
或是換個方向, Action 必須帶參數且透過 Model Binding 綁定, 而 ActionFilterAttribute 中維持不動, Action 直接操作綁定好的物件就好, 但這樣 ActionFilterAttribute 中還是把資源釋放掉了, 就設計的角度來看我們不希望在一個通用方法 (DemoActionFilterAttribute) 中對全域的資源做一次性的使用, 因為無法預期這個資源是否會再次被使用.

### (推薦) 直接讀內建的屬性
這個思考方向是, 那有沒有可能內建的 request 相關物件就有提供記錄著請求內容的 Stream 型別的屬性讓我們能直接讀取呢? 而內建的物件中的屬性, 我們可以合理預期不用手動去釋放相關的資源.  

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        var context = (HttpContextBase)actionContext.Request.Properties["MS_HttpContext"];
        var bytes = new byte[context.Request.InputStream.Length];
        context.Request.InputStream.Read(bytes, 0, bytes.Length);
        string body = Encoding.UTF8.GetString(bytes);
        // Logging here...
        logger.Info(body);
    }
}
```

這個做法看起來是最合理的, 一方面可以在 ActionFilter 中讀取請求內容, 不用手動釋放資源所以不會破壞它, 這樣各個 Action 中如果有需要也可以再次讀取.   

另外, 讀取 InputStream 的方式也可以搭配 StreamReader, 要特別帶入 leaveOpen 參數避免把 InputStream 關閉了, 比較麻煩, 例如下面的範例  

``` csharp
public class DemoActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        var context = (HttpContextBase)actionContext.Request.Properties["MS_HttpContext"];
        context.Request.InputStream.Seek(0, SeekOrigin.Begin);
        using (var sr = new StreamReader(context.Request.InputStream, Encoding.UTF8, true, 1024, true))
        {
            string body = sr.ReadToEnd();
            // Logging here...
            logger.Info(body);
        }
    }
}
```

### 結論
這是工作上遇到的問題, 研究一番後解是解了, 不過畢竟沒有完全了解底層的運作, 最後的解法是不是好的方法還很難說, 可能之後有空或是又踩到什麼相關問題再來仔細深究了.  
另外前面幾種解法雖然看起來都有些缺陷, 但也不全然是不好, 只是說在這個情境下不太適合而已.  

重置讀取位置這個行為在要讀取的 stream 已經被讀過時要做, 但前面的範例中, 某些情境下要讀取的 stream 不一定已經被讀過, 這個就看各人是習慣不論如何只要讀 stream 都重置讀取位置, 還是只在必要重置的時候重置了.

### 參考
[Web Api Request Content is empty in action filter](https://stackoverflow.com/questions/21351617/web-api-request-content-is-empty-in-action-filter)  
[Steam.CopyTo - Performance problems](http://vunvulearadu.blogspot.com/2013/04/steamcopyto-performance-problems.html)  
[Leave StreamReader without closing Stream](https://stackoverflow.com/questions/48564634/leave-streamreader-without-closing-stream)

---