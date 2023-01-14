---
title: 基於 ASP.NET Core WebAPI 讀取 Request / Response Body
date: 2021-06-26 00:59:20
categories:
- C#
- .NET Core
tags:
---

之前在 {% post_link webapi-get-request-body-safety %} 這篇文章有提到讀取 Request Body 的方式, Response 的部分其實大同小異, 且在 ASP.NET Core 中也是相同邏輯(即使實作可能不同), 但有個不同的地方是 ASP.NET Core 中的 request body 預設是讀完無法回捲的(也就是讀完後指標會在 stream 最後面, 無法重複讀取), 且 response body 是唯寫 (WriteOnly) 的, 所以無法讀取, 就需要一點小技巧.  

<!--more-->

### 程式碼
這邊直接實作在 Middleware 為例:  
``` csharp
public class RequestResponseMiddleware
{
    private readonly RequestDelegate _next;

    public RequestResponseMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext httpContext)
    {
        #region Request part

        var request = httpContext.Request;

        // 重要, 才能允許回捲 (request.Body.Seek(0, SeekOrigin.Begin))
        request.EnableBuffering();

        using var requestReader = new StreamReader(request.Body, leaveOpen: true);
        var body = await requestReader.ReadToEndAsync();
        request.Body.Seek(0, SeekOrigin.Begin);

        Debug.WriteLine(body);

        #endregion

        var response = httpContext.Response;

        var originalStream = response.Body;
        using (var readableBodyStream = new MemoryStream())
        {
            response.Body = readableBodyStream;

            await _next(httpContext);

            response.Body.Seek(0, SeekOrigin.Begin);

            using var responseReader = new StreamReader(response.Body, leaveOpen: true);
            var responseBody = await responseReader.ReadToEndAsync();

            Debug.WriteLine(responseBody);

            response.Body.Seek(0, SeekOrigin.Begin);

            await readableBodyStream.CopyToAsync(originalStream);
            // 即使沒有換回原本的物件, 得到的回傳仍然不會有錯, 猜測是回傳的內容是直接參考 response.Body 所指的物件,
            // 因此 response.Body 被換掉不會影響結果(反而上面的 stream copy 才是必要的), 但為了完整還是會把它換回來. 
            response.Body = originalStream;
        }

        #region Response part

        #endregion
    }
}
```
上面來說, Request 部分就 `request.EnableBuffering()` 最重要, 不然讀完回無法回捲, Response 的話 `await readableBodyStream.CopyToAsync(originalStream)` 和 `response.Body = originalStream` 這兩行算是比較特別的, 如註解中所述.  

### 結論
細節很多很難解釋, 只好貼 code 解決, 希望不會以後自己看不懂... 
