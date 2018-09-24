---
title: WebApi 2 中的 FilterAttribute 介紹與應用
date: 2018-09-23 14:30:04
categories:
- C#
- .NET
tags:
---

ASP.NET WebApi 2 提供了三個好用的 FilterAttribute 讓開發者可以擴充後將這些 FilterAttribute 套用在 Controller / Action 上面, 也可以在 WebApiConfig.cs 或 Global.asax 裡面註冊 , 並在一個 API request 生命週期中的特定時間被執行.  
這樣做的好處是可以將跟 API 主業務邏輯無關, 卻又是大部分 API 都要做的事收斂起來, 只需要將 Attribute 套用在正確的地方就能一體適用, 開發者就能更專注在 API 的主要商務邏輯層面.  

> 在了解 FilterAttribute 的運用之前, 必須先知道  Attribute 是什麼以及如何使用 Attribute.

<!--more-->

### 三種 FilterAttribute 介紹
ASP.NET WebApi 2 提供了三個好用的 FilterAttribute, 分別是 `AuthorizationFilterAttribute`,  `ActionFilterAttribute` 與 `ExceptionFilterAttribute` , 要注意的是命名空間都是 `System.Web.Http.Filters`, ASP.NET MVC 也有提供同名的 FilterAttribute, 但是命名空間不同, 內容也不盡相同.  

這三個 FilterAttribute 提供的功能各異:  
+ `AuthorizationFilterAttribute` 提供的是驗證相關的流程, 能在進 Action 之前就先執行相關的驗證方法.   
+ `ActionFilterAttribute` 會分別在 Action 執行前與執行完畢後執行相關方法.  
+ `ExceptionFilterAttribute` 在例外(unhandled exception)發生後執行.  

#### 基本用法
這三種 FilterAttribute 的使用方式都一樣, 這邊用 `ExceptionFilterAttribute` 來當作範例. 

1. **繼承對應的 FilterAttribute, 並覆寫父類別中的相關方法.** 
``` csharp
public class DemoExceptionFilterAttribute : ExceptionFilterAttribute
{
    public override void OnException(HttpActionExecutedContext actionExecutedContext)
    {
        // Call base method
        base.OnException(actionExecutedContext);

        // Do somthing, ex: logging exception
        // _logger.Info(actionExecutedContext.Exception);

        // Set response information
        actionExecutedContext.Response = new HttpResponseMessage(HttpStatusCode.InternalServerError);
    }
}

```

1. **套用 FilterAttribute.**
    + 套用到 Action
    ``` csharp
    public class DemoController : ApiController
    {
        [DemoExceptionFilter]
        public IEnumerable<string> Get()
        {
            throw new Exception();
        }
    }
    ```

    + 套用到特定 ApiController  
    ``` csharp
    [DemoExceptionFilter]
    public class DemoController : ApiController
    {
        public IEnumerable<string> Get()
        {
            throw new Exception();
        }
    }
    ```

    更進一步, 可以將這些 FilterAttribute 套用在一個共用的 BaseApiController 上, 只要繼承 BaseApiController, 就能直接套用.
    ``` csharp
    [DemoExceptionFilter]
    public class BaseApiController : ApiController
    {

    }

    public class DemoController : BaseApiController
    {
        public IEnumerable<string> Get()
        {
            throw new Exception();
        }
    }
    ```

    + 套用到所有 ApiController - 從 `~/App_Start/WebApiConfig.cs`
    ``` csharp
    public class DemoController : ApiController
    {
        public IEnumerable<string> Get()
        {
            throw new Exception();
        }
    }

    // ~/App_Start/WebApiConfig.cs
    public static void Register(HttpConfiguration config)
    {
        // ...

        config.Filters.Add(new DemoExceptionFilterAttribute());

        // ...
    }
    ```

    + 套用到所有 ApiController - 從`~/Global.asax`
    ``` csharp
    public class DemoController : ApiController
    {
        public IEnumerable<string> Get()
        {
            throw new Exception();
        }
    }

    // ~/Global.asax
    protected void Application_Start()
    {
        // ...

        GlobalConfiguration.Configuration.Filters.Add(new DemoExceptionFilterAttribute());

        // ...
    }
    ```

> 1. 單獨套用只能套用在 ApiController 的衍生類別或 Action 上, 如果是套用在其他方法(例如: xxxService.Do(), 或 this.Do()), 即使那個方法有被 ApiController 的 Action 呼叫到也是無法讓 FilterAttribute 運作的.  
> 2. 重複套用同一個 FilterAttribute 不會壞, 但因為註冊了兩次, 所以也會執行兩次, 所以應該要特別注意套用方式避免重複套用.  

#### 排除特定 Action 或 Controller 
如果大部分 API 都需要套用一個 FilterAttribute, 但有少數不套用, 應該要怎麼實作排除機制呢?  

1. **實作標記為排除的 Attribute**
``` csharp
public class IgnoreFilterAttribute : Attribute
{
    public Type FilterType { get; }

    public IgnoreFilterAttribute(Type filterType)
    {
        this.FilterType = filterType;
    }
}
```

1. **擴充 FilterAttribute 以支援排除功能**
``` csharp
public class DemoExceptionFilterAttribute : ExceptionFilterAttribute
{
    public override void OnException(HttpActionExecutedContext actionExecutedContext)
    {
        Func<IgnoreFilterAttribute, bool> ignoreCheck = (r) =>
        {
            // ignored type is one of...
            // 1. DemoExceptionFilterAttribute 
            // 2. Base types of DemoExceptionFilterAttribute
            return r.FilterType.IsAssignableFrom(typeof(DemoExceptionFilterAttribute));
        };

        var ignoredActions = actionExecutedContext
                                .ActionContext
                                .ActionDescriptor
                                .GetCustomAttributes<IgnoreFilterAttribute>()
                                .Any(ignoreCheck);

        var ignoredControllers = actionExecutedContext
                                    .ActionContext
                                    .ControllerContext
                                    .ControllerDescriptor
                                    .GetCustomAttributes<IgnoreFilterAttribute>()
                                    .Any(ignoreCheck);

        if (ignoredActions || ignoredControllers)
        {
            return;
        }

        // ...
    }
}

```

1. **套用標記為排除的 Attribute**
``` csharp
// apply to Controller
//[IgnoreFilter(typeof(DemoExceptionFilterAttribute))]
public class DemoController : ApiController
{
    // apply to Action
    [IgnoreFilter(typeof(DemoExceptionFilterAttribute))]
    public IEnumerable<string> Get()
    {
        throw new Exception();
    }
}
```

#### 執行順序
根據 [The ASP.NET Web API 2 HTTP Message Lifecycle in 43 Easy Steps](https://exceptionnotfound.net/the-asp-net-web-api-2-http-message-lifecycle-in-43-easy-steps-2/) 的說明加上 [ASP.NET WEB API 2: HTTP MESSAGE LIFECYLE](https://www.asp.net/media/4071077/aspnet-web-api-poster.pdf) 流程圖的內容來看, `AuthorizationFilterAttribute` 會先執行, 然後才是 `ActionFilterAttribute`, 而例外發生時會執行`ExceptionFilterAttribute`, 至於同種類的多個 FilterAttribute 或是一個 FilterAttribute 中的不同方法執行的順序, 可以從下面的程式碼觀察出結果.  

首先, 需要 `AuthorizationFilterAttribute` 和 `ActionFilterAttribute` 各兩組, 另外一組 `ExceptionFilterAttribute`, 接著把這些 FilterAttribute 放在 WebApi 專案預設的 Action 上, 且 `ExceptionFilterAttribute` 刻意插在中間, 而 Action 就簡單的拋出一個例外.  

``` csharp
public class ValuesController : ApiController
{
    // GET api/values
    [MyAuth2]
    [MyAuth1]
    [MyAction1]
    [MyException1]
    [MyAction2]
    public IEnumerable<string> Get()
    {
        throw new Exception();
    }
}
```

實作部分所有 FilterAttribute 並覆寫所有能覆寫的方法, 內容只是單純寫個簡單的歷程記錄到 `RouteTracer.Routes` 中, 但`ExceptionFilterAttribute` 需要另外負責回傳正確的狀態與 `RouteTracer.Routes` 內容, 才能順利觀察執行順序. 

``` csharp
public class MyAuth1 : AuthorizationFilterAttribute
{
    public override void OnAuthorization(HttpActionContext actionContext)
    {
        RouteTracer.Routes.Add("MyAuth1.OnAuthorization");
        base.OnAuthorization(actionContext);
    }

    public override Task OnAuthorizationAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAuth1.OnAuthorizationAsync");
        return base.OnAuthorizationAsync(actionContext, cancellationToken);
    }
}

public class MyAuth2 : AuthorizationFilterAttribute
{
    public override void OnAuthorization(HttpActionContext actionContext)
    {
        RouteTracer.Routes.Add("MyAuth2.OnAuthorization");
        base.OnAuthorization(actionContext);
    }

    public override Task OnAuthorizationAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAuth2.OnAuthorizationAsync");
        return base.OnAuthorizationAsync(actionContext, cancellationToken);
    }
}

public class MyAction1 : ActionFilterAttribute
{
    public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
    {
        RouteTracer.Routes.Add("MyAction1.OnActionExecuted");
        base.OnActionExecuted(actionExecutedContext);
    }

    public override Task OnActionExecutedAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAction1.OnActionExecutedAsync");
        return base.OnActionExecutedAsync(actionExecutedContext, cancellationToken);
    }

    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        RouteTracer.Routes.Add("MyAction1.OnActionExecuting");
        base.OnActionExecuting(actionContext);
    }

    public override Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAction1.OnActionExecutingAsync");
        return base.OnActionExecutingAsync(actionContext, cancellationToken);
    }
}

public class MyAction2 : ActionFilterAttribute
{
    public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
    {
        RouteTracer.Routes.Add("MyAction2.OnActionExecuted");
        base.OnActionExecuted(actionExecutedContext);
    }

    public override Task OnActionExecutedAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAction2.OnActionExecutedAsync");
        return base.OnActionExecutedAsync(actionExecutedContext, cancellationToken);
    }

    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        RouteTracer.Routes.Add("MyAction2.OnActionExecuting");
        base.OnActionExecuting(actionContext);
    }

    public override Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
        RouteTracer.Routes.Add("MyAction2.OnActionExecutingAsync");
        return base.OnActionExecutingAsync(actionContext, cancellationToken);
    }
}

public class MyException1 : ExceptionFilterAttribute
{
    public override void OnException(HttpActionExecutedContext actionExecutedContext)
    {
        RouteTracer.Routes.Add("MyException1.OnException");
        base.OnException(actionExecutedContext);
        actionExecutedContext.Response = new HttpResponseMessage(HttpStatusCode.OK);
        actionExecutedContext.Response.Content = new ObjectContent<List<string>>(RouteTracer.Routes, new XmlMediaTypeFormatter());
    }
}
```

實際運作後頁面顯示結果如下, 可以看出各種 FilterAttribute 以及其中的方法的執行順序.  
 
``` xml
<ArrayOfstring xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/2003/10/Serialization/Arrays">
    <string>MyAuth2.OnAuthorizationAsync</string>
    <string>MyAuth2.OnAuthorization</string>
    <string>MyAuth1.OnAuthorizationAsync</string>
    <string>MyAuth1.OnAuthorization</string>
    <string>MyAction1.OnActionExecutingAsync</string>
    <string>MyAction1.OnActionExecuting</string>
    <string>MyAction2.OnActionExecutingAsync</string>
    <string>MyAction2.OnActionExecuting</string>
    <string>MyAction2.OnActionExecutedAsync</string>
    <string>MyAction2.OnActionExecuted</string>
    <string>MyAction1.OnActionExecutedAsync</string>
    <string>MyAction1.OnActionExecuted</string>
    <string>MyException1.OnException</string>
</ArrayOfstring>
```

就執行順序的這部分, 總結來說  
+ `AuthorizationFilterAttribute` 先於 `ActionFilterAttribute` 執行 (從流程圖上來看合理).  
+ 多個同種類的 FilterAttribute 實作, 會依照套用的順序執行, 但 `OnActionExecuted` 與 `OnActionExecutedAsync` 是反序 (從流程圖上來看也合理, 因為 OnActionExecuted 系列方法是在 Action 執行後要 response 時才執行).  
+ 如果有例外發生的話, `ExceptionFilterAttribute` 會在最後才執行.  
+ 一個 FilterAttribute 內部的方法執行順序看起來是非同步方法先於同步方法.  

> 1. 單一測試就下結論其實不穩妥, 所以我後來又做了幾次不一樣的排序實驗, 看起來結果是符合推測的, 但無法完全保證.  
> 2. 有些特性可能會隨著版本的變遷改變, 如果實際專案要用, 又真的很在意這些順序或其他細節的話, 還是要再就實際狀況測試一輪.

### 結論
想像一下, 當有一天團隊要導入 log 收集與分析工具, 需要統一 log 的格式時, 如果當初有用 FilterAtrribute 來印通用訊息就可以省下非常多瑣碎的工, 而例外處理也是同理, 且能避免到處都是為了抓 unhandled exception 而寫的 `try-catch` 語句.  

### 參考
[How to disable a global filter in ASP.Net MVC selectively](https://stackoverflow.com/questions/9953760/how-to-disable-a-global-filter-in-asp-net-mvc-selectively)  
[ASP.NET Web API Exception Filter](https://www.huanlintalk.com/2013/01/aspnet-web-api-exception-filter.html)  
[Exclude A Filter](http://blogs.microsoft.co.il/oric/2011/10/28/exclude-a-filter/)  
[The ASP.NET Web API 2 HTTP Message Lifecycle in 43 Easy Steps](https://exceptionnotfound.net/the-asp-net-web-api-2-http-message-lifecycle-in-43-easy-steps-2/)  
[ASP.NET WEB API 2: HTTP MESSAGE LIFECYLE](https://www.asp.net/media/4071077/aspnet-web-api-poster.pdf)