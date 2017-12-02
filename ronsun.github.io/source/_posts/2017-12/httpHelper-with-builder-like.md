---
title: 如何避免大量參數 - 以HttpHelper為例
date: 2017-12-02 23:54:45
categories:
- DesignPatterns
---

在設計 API 的時候, 常常會被參數過多所困擾著, 因為當方法有著過多參數時, 使用的時候容易眼花, 而需要增減參數時也很不方便, 這邊以常見的 HttpHelper 為例來說明。  

<!--more-->

一般來說, 如果專案只需要跟內部的 API 溝通的話, 其實用 HttpClient 來實作既方便又快速, 但是大量外部廠商對接溝通的時候可能就不太適合, 因為每家廠商對接的方式都不一樣, 所以目前的專案是用 WebRequest 來實作。   

---

#### 擁有大量參數的方法

``` csharp

public string Post(string url, string content, Encoding encoding, int contentType, string proxyAddress, string proxyUserName, string proxyPassword)
{
    var request = WebRequest.CreateHttp(url);
    request.ContentType = contentType;
    request.Method = WebRequestMethods.Http.Post;

    var contentBytes = encoding.GetBytes(content);
    request.ContentLength = contentBytes.Count();

    //設定proxy

    //發送request, 取得並回傳response
}

```

需求的一開始, 參數可能只有 url 跟 content, 但是當對接廠商越來越多的時候, 有人要求只能用特定編碼(UTF8, GBK...), 資料格式不同時 content type 也要不同, 因為網路環境限制有時候要走 proxy 等等的, 這時候就會出現大量參數, 引發一些問題:  
  + 增加參數, 那所有呼叫端的程式都要改, 不難, 但是麻煩
  + 呼叫端改好了, 為了避免帶錯值, 所以要測試一輪, 也不難, 但是很花時間  

那就有改善的空間了  

---

#### 選擇性引數(Optional Arguments) 

> 註: 口語上大多稱選擇性參數(Optional Parameters), 不過MSDN是用選擇性引數(Optional Arguments)  
  
> 註: 參數和引數意義上是不同的  

``` csharp
public string Post(string url, string content = "", Encoding encoding = null, int contentType = "application/x-www-form-urlencoded", string proxyAddress = "", string proxyUserName = "", string proxyPassword = "")
{
    if(encoding == null)
    {
        encoding == Encoding.UTF8;
    }

    var request = WebRequest.CreateHttp(url);
    request.ContentType = contentType;
    request.Method = WebRequestMethods.Http.Post;

    var contentBytes = encoding.GetBytes(content);
    request.ContentLength = contentBytes.Count();

    //設定proxy

    //發送request, 取得並回傳response
}
```

這樣用的好處是, 當需要增加非必要參數的時候, 可以直接加在參數列最後面, 並給他一個預設值, 那至少加參數的時候不用像尋寶一樣到處去找呼叫端修改了, 降低了手誤產生bug的風險, 也可以把測試專注在這個方法內就好。   

但是, 還是有缺點:
+ 參數列還是超長
+ 參數預設值只能是編譯時期就決定好的常數, 所以Encoding這類參數, 必須預設為null, 然後另外在方法內判斷賦予預設值
+ 新的參數如果不是加在最後面, 在某些情境下會有問題

> 註:  
> 基於以下方法, 呼叫 `foo(0, "B");` 時代表的是 `foo(0, b = "B")`  
> 但是如果插了一個參數在b之前, 而型別和b一樣, 方法簽章變成
> `public void foo(int a, string c = "", string b = "")`
> 這時候呼叫 `foo(0, "B");`代表的就會是`foo(0, c = "B")`
> **因此這邊會建議在呼叫帶有選擇性引數的方法時, 採用具名方式呼叫, 避免上面範例中的誤用**

所以還是有改善的空間  

---

#### 把所有參數包成一個condition物件

``` csharp
public class Condition
{
    public string Url { get; set; }
    public string Content { get; set; }
    public Encoding Cncoding { get; set; } = Encoding.UTF8;
    public string ContentType { get; set; } = "application/x-www-form-urlencoded";

    //其他省略
}

public string Post(Condition condition)
{
    //基於condition準備request相關物件

    //設定proxy

    //發送request, 取得並回傳response
}

```

好多了, 這次的改善避免了之前參數過長, 預設值也不受限於常數, 同時當需要增加非必填的參數時, 只需要修改 condition 物件並設好預設值, 呼叫端基本上不用有任何修改。  

但是真要在吹毛求疵的話, 還是有一個缺點: **API使用者無法一眼看出那些參數是必填**.  

我個人的習慣是, 如果不想讓使用者誤用, 那就從技術層面阻止他, 誘導使用者正確使用, 所以接下來接下來  

---
 
#### condition 物件的變形  

直接先上完整版, 下面會細說

``` csharp
public class Condition
{
    public string Url { get; private set; }

    public string Content { get; private set; }

    public Encoding Encoding { get; private set; } = Encoding.UTF8;
    
    public string ContentType { get; private set; } = "application/x-www-form-urlencoded";

    public IWebProxy Proxy { get; private set; } = null;

    private Condition() { }
    
    public static Condition Create(string url)
    {
        var reqParams = new Condition()
        {
            Url = url
        };

        return reqParams;
    }

    public Condition WithContentType(string contentType)
    {
        ContentType = contentType;
        return this;
    }
    
    public Condition WithContent(string content)
    {
        Content = content;
        return this;
    }
    
    public Condition WithEncoding(Encoding encoding)
    {
        Encoding = encoding;
        return this;
    }
    
    public Condition WithProxy()
    {
        //default proxy
        IWebProxy defaultProxy = WebRequest.GetSystemWebProxy();
        defaultProxy.Credentials = CredentialCache.DefaultCredentials;
        Proxy = defaultProxy;
        return this;
    }
    
    public Condition WithProxy(string address, string userName, string password)
    {
        IWebProxy webProxy = new WebProxy(address);
        webProxy.Credentials = new NetworkCredential(userName, password);
        Proxy = webProxy;
        return this;
    }
}
```

這樣做有幾個重點與目的  
+ `private Condition() { }` 把預設的建構子設成私有, 然後透過有著**必填參數**的靜態方法 `Create(string url)` 來建立物件, 目的是讓使用者沒有機會漏填必填的參數
+ 使用者建立物件後可以直接呼叫 `WithXXXX(...)` 方法設定非必填的參數, 具體呼叫範例: `Condition.Create("http://sample.com").WithContent("<xml>balalala</xml>").WithProxy();`, 這樣的方法鏈非常方便, 是來自於 [Fluent Interface](https://en.wikipedia.org/wiki/Fluent_interface) 的概念
+ 這樣 condition 組好直接作為 `Post(Condition conditon)` 的參數就可以了  

這種方式是從建構者模式和 Fluent Interface 變形而來的, 建構者模式是把**建構者** 和 **被建構者** 分成兩個物件, 只不過對於一個只是要整合大量參數的需求來說, 要特別建立 Condition 和 ConditionBuilder 兩個物件是有點太多了, 所以稍微變化一下簡化他的複雜度, 網路上也有結合建構者模式和 Fluent Interface 做成 [Fluent Builder](https://code-maze.com/builder-design-pattern/) 的例子.  
**但是!!**  
**但是!!**  
**但是!!**  

如果不是非常複雜的情境的話, 這樣是有點過度設計了, 雖然呼叫起來很方便, 但會使得 Condition 變得比較複雜, 如果系統中到處都是這種東西的話其實會提高後續的維護門檻的.

---

#### (推薦)只將選填參數包成 Condition 物件

``` csharp
public class RequestOptions
{
    public Encoding Cncoding { get; set; } = Encoding.UTF8;
    public string ContentType { get; set; } = "application/x-www-form-urlencoded";

    //其他省略
}

public string Post(string url, string content, RequestOptions options = null)
{
    if(options == null)
    {
        options = new RequestOptions();
    }

    //基於options準備request相關物件

    //設定proxy

    //發送request, 取得並回傳response
}

```

**這個版本是我目前最推薦的**, 一方面他減少了參數的數量, 另一方面把必填與選填參數分開, 避免使用者誤用, 也夠簡單好懂.


#### 後記  
其實自己在寫東西常常弄到過度設計的窘境, 最近在練習怎麼樣才能把程式寫得剛好, 又能在易用, 好維護與好擴充中間找到平衡點(但好難XDD).

#### 參考資料  
[how-to-avoid-too-many-parameters-problem-in-api-design](https://stackoverflow.com/questions/6239373/how-to-avoid-too-many-parameters-problem-in-api-design)  
[Builder Design Pattern and Fluent Builder](https://code-maze.com/builder-design-pattern/)
