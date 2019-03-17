---
title: Web API 回傳資料的方式
date: 2019-03-17 14:38:52
categories:
- C#
- .NET
tags:
---

這篇是要稍微記一下 Web API 回傳資料的幾種方式與格式, 會想要記這一篇是因為以前碰過的 Web API 專案都有統一的回傳 (response) 格式, 但是最近碰到一些專案的 Web API 在回傳資料時是沒有規範的, 例如直接就 `return true`, 所以就稍微整理一下回傳資料的幾種方式與優缺點.  

<!--more-->

### 直接回傳物件
這個就是上面說的, 直接像 `return anyObject` 這樣回傳, 如下面的程式碼

``` csharp
public bool Get()
{
    return true;
}

public DemoModel Get()
{
    var data = new DemoModel()
    {
        Name = "Ron",
        Age = 18
    };

    return data;
}

public Dictionary<string,string> Get()
{
    var data = new Dictionary<string, string>()
    {
        ["Name"] = "Ron",
        ["Age"] = "18"
    };

    return data;
}
```

#### 優缺點
**優點**  
+ 簡單直覺又方便

**缺點**
+ 凌亂不統一, 回傳的資料可能出現原生型別, model 或是其他各種型別
+ 只回傳資料, 沒有任何狀態類的欄位與 http status, 當 client 端收到空物件或例外時難以判斷問題  
+ 後期需要加狀態時很麻煩, 例如每支 API 各自加狀態欄位容易亂, 如果要抽共用還要考慮資料的結構或階層被改變對客戶端的影響

#### 使用時機
這種做法適合用在
+ 簡單的小專案, 複雜度小就沒太大影響
+ Web API 只提供內部使用, client 跟 API 是同一組人開發的, 狀態類欄位的必要性不大

### 套用共用回傳物件與狀態
同時考慮回傳資料, http status 與例外內容三種情況, 如下

> `IHttpActionResult` 是 Web API 2 的方式

``` csharp
public class ResponseModel<T>
{
    public string Code { get; set; } = "0000";

    public string Message { get; set; } = "Success";

    public T Data { get; set; }
}

// Web API
public HttpResponseMessage Get(bool isError)
{
    var responseModel = new ResponseModel<DemoModel>();

    if (isError)
    {
        responseModel.Code = "0001";
        responseModel.Message = "Prameter 'isError' should be false.";
        var response = Request.CreateResponse(HttpStatusCode.BadRequest, responseModel);
        throw new HttpResponseException(response);
    }

    responseModel.Data = new DemoModel()
    {
        Name = "Ron",
        Age = 18
    };

    return Request.CreateResponse(HttpStatusCode.OK, responseModel);
}

// Web API 2
public IHttpActionResult Get(bool isError)
{
    var responseModel = new ResponseModel<DemoModel>();

    if (isError)
    {
        responseModel.Code = "0001";
        responseModel.Message = "Prameter 'isError' should be false.";
        return Content(HttpStatusCode.BadRequest, responseModel);
    }

    responseModel.Data = new DemoModel()
    {
        Name = "Ron",
        Age = 18
    };

    return Ok(responseModel);
}
```

#### 特別的 HttpResponseException
上面例子中, `HttpResponseException` 是一個很特別的例外, 這個例外不會被 `ExceptionFilterAttribute`, `ExceptionHandler` 或是 `ExceptionLogger` 所攔截, 所以在 service 層中使用 `HttpResponseException` 是一個很好的選項, 既能直接返回 http status 和內容, 又不會被全域例外處理機制攔截, 像是這樣 `throw new HttpResponseException(response)`.  

順帶一提, `ExceptionHandler` 和 `ExceptionLogger` 是實測的時候才發現會忽略 `HttpResponseException`, [官方文件只提到 `ActionFilterAttribute` 的部分](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling), 節錄如下:  

> An exception filter is executed when a controller method throws any unhandled exception that is not an HttpResponseException exception. The HttpResponseException type is a special case, because it is designed specifically for returning an HTTP response.

#### 優缺點
**優點**
+ 明確回傳 http status, 對 client 端來說遇到問題容易釐清方向
+ 細部狀態碼 (Code) 與訊息 (Message) 方便 client 端對接與使用

**缺點**
+ 比較麻煩, 需要花時間了解並選擇適合的 http status, 也需要定義細部狀態碼 (Code) 與訊息 (Message)

#### 共用回傳物件的幾種設計
回傳物件的共用部分要怎麼設計也是有幾種方式

**泛型版**  
包一層 `ResponseModel<T>`, 可以直接套用到所有需要回傳物件的 API 上, 但是因為多包了一層, 序列化的結果會多一層, 如果套用在舊的 API 上會使得回傳資料的階層被改變, 會影響所有 client 端.  

``` csharp
public class ResponseModel<T>
{
    public string Code { get; set; } = "0000";

    public string Message { get; set; } = "Success";

    public T Data { get; set; }
}
```

**不用泛型**  
沒有泛型, `ResponseModel` 讓其他要回傳的 model 繼承或包含, 如果套用在舊的 API 上, 在序列化後只會增加欄位, 不會改變舊有欄位的階層, 對現存 API 的擴展很友善, 但是相對於泛型版比較不好用, 因為即使只是要回傳一個基本型別(bool / string) 也需要建立 model.

``` csharp
public class ResponseModel
{
    public string Code { get; set; } = "0000";

    public string Message { get; set; } = "Success";
}
```

回傳物件型別有兩種使用 `ResponseModel` 的方法, 繼承或包含, 序列化結果不同, 視情境或團隊共識使用.

``` csharp
// 繼承
public class DemoModel : ResponseModel
{
    public string Name { get; set; }

    public int Age { get; set; }
}

// 包含
public class DemoModel
{
    public string Name { get; set; }

    public int Age { get; set; }

    public ResponseModel Error {get; set; }
}
```

#### 使用時機
這種做法適合用在
+ 邏輯複雜或是有各種驗證狀態與結果的 API
+ 需要對外開放的 API (跨部門 / 公開 API)

### 結論
雖然稍微整理一下幾種做法與適用情境, 不過實際運用不太會照抄, 還是要視情況做細部的調整, 就跟設計模式在使用的時候常常需要配合專案需求變形過一樣.  

### 參考
[Best practice to return errors in ASP.NET Web API](https://stackoverflow.com/questions/10732644/best-practice-to-return-errors-in-asp-net-web-api)  
[Exception Handling in ASP.NET Web API](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling)  