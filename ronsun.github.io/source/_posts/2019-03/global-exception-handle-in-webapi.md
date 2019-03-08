---
title: Web API 中的全域錯誤處理
date: 2019-03-08 21:27:34
categories:
- C#
- .NET
tags:
---

在 Web API 中, 例外處理是一定會遇到的問題, 以前常常見到一些專案為了怕出現 unhandled exception 就地毯式的將所有方法都加上 try-catch, 但是這樣使用 try-catch 會有很大的副作用, 一來無差別的吃掉所有例外會導致程式不易除錯, 二來到處散布的 try-catch 和相關的 log 在維護上的成本也很高, 尤其是大型專案更是明顯, 第三也會使得商務邏輯跟這些例外處理混雜在一起 (try-catch 不是不能用, 但是他是有使用時機的).  

而 ASP.NET Web API 2 中其實提供了很多種全域的例外處理方式, 例如: `ExceptionFilterAttribute`, `ExceptionLogger`, `ExceptionHandler`, 能將攔截未處理的例外的任務抽離出來, 讓我們開發時能更專注在重要的商務邏輯上, 這篇主要紀錄一下這三種全域例外處理的方式與差別.  

<!--more-->

### ExceptionFilterAttribute

 在 {% post_link filters-of-webapi2 %} 這篇文章中已經有提到 FilterAttribute 的用法, 但是 `ExceptionFilterAttribute` 他在整個 [Web API 的生命週期](https://www.asp.net/media/4071077/aspnet-web-api-poster.pdf)中是比較後面的, 所以某些階段所引發的例外他無法捕捉到, [官方也明確指出了這個特性](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/web-api-global-error-handling), 節錄如下:

> Some unhandled exceptions can be processed via exception filters, but there are 
> a number of cases that [exception filters](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling) can't handle. For example:
> 1. Exceptions thrown from controller constructors.
> 1. Exceptions thrown from message handlers.
> 1. Exceptions thrown during routing.
> 1. Exceptions thrown during response content serialization.
> 1. We want to provide a simple, consistent way to log and handle (where possible) 
> these exceptions.

`ExceptionFilterAttribute` 是個不錯的選項, 但受限於生命週期, 所以攔截的範圍沒那麼廣.  

### ExceptionLogger / ExceptionHandler

`ExceptionLogger` 和 `ExceptionHandler` 能攔截 `ExceptionFilterAttribute` 所攔截不到的例外, 下面分別是 `ExceptionLogger` 和 `ExceptionHandler` 的實作方式.   

#### 實作方式

``` csharp
public class DemoExceptionHandler : ExceptionHandler
{
	public override void Handle(ExceptionHandlerContext context)
	{
		Debug.WriteLine("======== DemoExceptionHandler =======");

		context.Result = new InternalServerErrorResult(context.Request);
	}
}

public class DemoExceptionLogger : ExceptionLogger
{
	public override void Log(ExceptionLoggerContext context)
	{
		Debug.WriteLine("======== DemoExceptionLogger =======");
	}
}
```

實作後要在 WebApiConfig.cs 中註冊.  

``` csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		// skip...

		config.Services.Replace(typeof(IExceptionHandler), new DemoExceptionHandler()
		config.Services.Add(typeof(IExceptionLogger), new DemoExceptionLogger()););

		// skip...
	}
}
```

另外有幾個要注意的點:
+ `ExceptionHandler` 只能有一個, 所以呼叫 `config.Services.Replace()` 
+ `ExceptionLogger` 可以有多個, 所以呼叫 `config.Services.Add()`, 存在多個 `ExceptionLogger` 時, 先加入的先執行  
+ `ExceptionLogger` 會比 `ExceptionHandler` 先執行
+ `ExceptionLogger` 也可以呼叫 `config.Services.Replace()`, 不管之前加了幾個 `ExceptionLogger`, `config.Services.Replace()` 會將全部都取代成一個, 所以他不是換掉某一個而已 

#### 差異

`ExceptionLogger` 是用於紀錄攔截到的例外, `ExceptionHandler` 是用於攔截到錯誤後的處理, 所以 `ExceptionHandlerContext` 中多了一個 `public IHttpActionResult Result { get; set; }` 屬性, 讓我們可以決定攔截到錯誤後要回傳什麼給客戶端, 這在上面程式碼中也可以看出差異.  

如果同時要紀錄攔截到的錯誤並回傳特定內容給客戶端, 目前我是只用 `ExceptionHandler` 處理, 沒有同時實作 `ExceptionLogger` 和 `ExceptionHandler`.  

### 結論
以前都只用 `ExceptionFilterAttribute` 來攔截錯誤, 也沒有特別去找其他做法, 剛好跟別人聊天時有提到全域例外處理跟生命週期的話題才被提醒有這個差別, 意外賺到一個之前都不知道的好東西.  

### 參考
[Global Error Handling in ASP.NET Web API 2](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/web-api-global-error-handling)  
[Exception Handling in ASP.NET Web API](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling)  

---