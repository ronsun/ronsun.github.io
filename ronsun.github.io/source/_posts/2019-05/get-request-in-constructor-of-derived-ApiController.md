---
title: 在建構子中取得 WebAPI 的 request 內容
date: 2019-05-26 01:14:48
categories:
- C#
- .NET
tags:
---

事情是這樣的, 有一天我需要在某個共用的 BaseApiController 中取得 request 中的內容並做一些操作, 所以寫下了像是這樣的內容, 然後 ~~他就死掉了~~ 就出現 `NullReferenceException` 了.  

``` csharp
public class BaseApiController : ApiController
{
	public BaseApiController()
	{
		var request = Request;
		// do someting from request, ex: read headers
		var myHeader = request.Headers.GetValues("MyHeader");
	}
}
```

<!--more-->

### 改用 HttpContext.Current.Request
發生這個問題其實一開始有點驚訝, 畢竟以前在 Action 中操作 `ApiController.Request` 都非常順利, 接著很快的在網路上查到了[一個相同的提問]((https://stackoverflow.com/questions/20428305/request-is-always-null-in-web-api), 表示改用 `HttpContext.Current.Request` 就好了, 然後問題就解決了.  

根據討論串的說法, `ApiController.Request` 不能在建構子中使用, 因為在這個階段他還沒被初始化, 所以必然是 null.

### 兩者有什麼差別?
接著試著了解一下 `HttpContext.Current.Request` 和 `ApiController.Request` 之間的差別.  

首先可以發現 `ApiController.Request` 回傳了一個 HttpRequestMessage 物件, 且這個屬性是可讀寫的, 而 `HttpContext.Current.Request` 回傳的是 HttpRequest 物件, 但這個屬性是唯讀的, 另外從下面的原始碼可以看到 HttpRequestMessage 是回傳 `ControllerContext.Request` 且往上可以追朔到 ActionContext, 由此"推測" `ApiController.Request` 比較適合在 Action 中使用.  

> 推測跟框架的生命週期與相關成員的初始化時間有關, 但要驗證的話要去爬 code, 目前沒這個需要就先偷懶一下了, 至於 HttpRequest 和 HttpRequestMessage 的差別在 [HttpRequest vs HttpRequestMessage vs HttpRequestBase](https://stackoverflow.com/questions/15718468/httprequest-vs-httprequestmessage-vs-httprequestbase) 這篇文章有稍微提到, 不過沒講得太深入.  

``` csharp
// copy from https://github.com/aspnet/AspNetWebStack/blob/master/src/System.Web.Http/ApiController.cs
public HttpRequestMessage Request
{
	get
	{
		return ControllerContext.Request;
	}
	set
	{
		if (value == null)
		{
			throw Error.PropertyNull();
		}

		HttpRequestContext contextOnRequest = value.GetRequestContext();
		HttpRequestContext contextOnController = RequestContext;

		if (contextOnRequest != null && contextOnRequest != contextOnController)
		{
			// Prevent unit testers from setting conflicting requests contexts.
			throw new InvalidOperationException(SRResources.RequestContextConflict);
		}

		ControllerContext.Request = value;
		value.SetRequestContext(contextOnController);

		RequestBackedHttpRequestContext requestBackedContext =
			contextOnController as RequestBackedHttpRequestContext;

		if (requestBackedContext != null)
		{
			requestBackedContext.Request = value;
		}
	}
}
```

### 結論
會碰到這個大概就是對於相似的成員之間的差異與生命週期不了解, 可能要找個時間好好了解一下了. ~~不過不知道這個 `//TODO` 會拖多久就是~~.  

### 參考
[Request is always null in web api?](https://stackoverflow.com/questions/20428305/request-is-always-null-in-web-api)  
[HttpRequest vs HttpRequestMessage vs HttpRequestBase](https://stackoverflow.com/questions/15718468/httprequest-vs-httprequestmessage-vs-httprequestbase)