---
title: 透過 GET 發送 HTTP Request 的時候 Uri 過長的問題
date: 2023-01-14 02:07:23
categories:
- C#
- .NET Core
tags:
---

很多時候我們需要從客戶端從透過 HTTP GET 發送 Reqeust 給服務端，但在之前的一個 ASP.NET Core 的專案中卻因為 Uri 過長而出現了 `Invalid URI: The uri string is too long` 這樣的錯誤訊息。  

這個問題網路上幾乎都是改 Web Service (例如: IIS) 的設定來解決，這是最簡單方便的方法，但如果伺服器很多或是需要經常擴充伺服器時就會造成不小的困擾，即使伺服器不多也可能因為這些瑣碎的設定造成維護上的不方便，所以這邊提供另外一種解決方案。

<!--more-->

### X-Http-Method-Override
#### 緣起
X-Http-Method-Override 這個 Header 典型的用法是用於用戶端因為各種限制而無法配合服務端的要求使用特定 HTTP Method 時，以[這篇文章](https://www.hanselman.com/blog/http-put-or-delete-not-allowed-use-xhttpmethodoverride-for-your-rest-service-with-aspnet-web-api)為例， 服務端要求的是 `PUT /api/Person/4`，但客戶端無法配合時，就可以改用 `POST /api/Person/4` 搭配 X-Http-Method-Override 這個 Header (值為 PUT) 來改變實際上的 HTTP Method，這可以不需要遷就客戶端而讓服務端的設計保持一致且合理。  

利用這個特性，我們可以試著讓呼叫端改用 HTTP POST 來發送以避開 Web Server 對 Uri 的長度限制。

> 同理，對 QueryString 的長度限制也適用。

#### 實做
首先，從[這篇文章](https://vnextcoder.wordpress.com/2018/06/05/how-to-allow-http-method-override-dotnet-core/)我們可以知道 ASP.NET Core 有提供相關的功能來支援 X-Http-Method-Override，但仔細看 [UseHttpMethodOverride](https://github.com/dotnet/aspnetcore/blob/main/src/Middleware/HttpOverrides/src/HttpMethodOverrideExtensions.cs#L20) 和 [HttpMethodOverrideMiddleware](https://github.com/dotnet/aspnetcore/blob/main/src/Middleware/HttpOverrides/src/HttpMethodOverrideMiddleware.cs#L13) 的實做可以發現這並不足以滿足我們的需求，HttpMethodOverrideMiddleware 的實做只包含改變 HTTP Method 但不包含處理 Body 或 QueryString。

但至少我們知道 ASP.NET Core 對於 X-Http-Method-Override 的支援是透過 Middleware 來完成的，這樣我們就可以自己寫 Middleware 來達到想要的效果。

**建立另外一個 Middleware 來處理 Body 的部分: **  
框架內建的部份我們就沿用就好，所以自製的 Middleware 只需要處理 Body。
``` csharp
public class HttpGetOverrideMiddleware
{
    private const string XHttpMethodOverride = "X-Http-Method-Override";

    private readonly RequestDelegate _next;

    public HttpGetOverrideMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public Task Invoke(HttpContext context)
    {
        var xHttpMethodOverrideValue = context.Request.Headers[XHttpMethodOverride];
        if (HttpMethods.IsGet(xHttpMethodOverrideValue))
        {
            /* pseudo code
            var body = ReadFromRequestBody();
            
            string anotherQueryString = string.Empty();
            if (isJson)
            {
                anotherQueryString = ParseBodyFromJsonToQueryString();
            }
            else if (isXml)
            {
                anotherQueryString = ParseBodyFromXmlToQueryString();
            }
            else if (whatever)
            {
                anotherQueryString = ParseBodyFromWhateverToQueryString();
            }
            
            context.Request.QueryString += anotherQueryString;
            
            ClearBody(); // if possible
            */
        }

        return _next(context);
    }
}
```

**註冊並使用: **  
``` csharp
public class Startup
{        
    public void Configure(IApplicationBuilder app)
    {
        app.UseHttpMethodOverride();
        app.UseMiddleware<HttpGetOverrideMiddleware>();
    }
}
```

如上所示，在 Middleware 那一層將 Body 轉換成 QueryString 再繼續，這樣一來 Web API 的規格仍然是合理的 HTTP GET， 呼叫端也可以透過 HTTP POST 加上 X-Http-Method-Override 來呼叫到只支援 HTTP GET 的 API。

**注意事項: **  
+ 將 Body 轉換為 QueryString 的過程有很多瑣碎的細節要考慮
    - 如果原本的 Uri 就包含 QueryString 的話(不合理，但技術上可能發生)不能覆蓋掉
    - Body 讀取後要不要把內容清空?
    - 不清空 Body 的話要不要允許後面的程式能重複讀取 Body? 這會影響 Body 讀取後要不要回捲
    - Body 資料無法用 QueryString
+ 將 Body 轉換為 QueryString 後整個 Uri 會很長，後面的程式直接操作時要小心避免因過長拋出例外，例如:
`new Uri(string)` 就有機會因為長度太長而拋出例外。
+ Startup 中呼叫 `UseHttpMethodOverride()` 和註冊 `HttpGetOverrideMiddleware` 的順序，誰先誰後比較合理?

### 結論
這個做法看起來有點暴力，但因為 ASP.NET Core 也是用這樣的方式支援 X-Http-Method-Override 的，所以就跟隨原有的做法下去擴充了。  

另外值得一提的是， ASP.NET Core 的很多功能 (例如: Health checks) 背後其實都是封裝 Middleware 後提供一個較易用或可讀性較高的 API，知道這點後也能了解很多類似的功能都可以用 Middleware 來實現，只要小心不要弄錯執行順序的話是很好用的。

### 參考
[HTTP PUT or DELETE not allowed? Use X-HTTP-Method-Override for your REST Service with ASP.NET Web API](https://www.hanselman.com/blog/http-put-or-delete-not-allowed-use-xhttpmethodoverride-for-your-rest-service-with-aspnet-web-api)  

[HttpMethodOverrideMiddleware.cs](https://github.com/dotnet/aspnetcore/blob/main/src/Middleware/HttpOverrides/src/HttpMethodOverrideMiddleware.cs#L13)  

[HttpMethodOverrideExtensions.cs](https://github.com/dotnet/aspnetcore/blob/main/src/Middleware/HttpOverrides/src/HttpMethodOverrideExtensions.cs#L20)