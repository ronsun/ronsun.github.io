---
title: 使用 using 以及當例外從 finally 區塊拋出時的問題
date: 2023-03-11 23:28:28
categories:
- C#
tags:
---

當我們想要使用實作 `IDisposable` 的型別時，`using` 關鍵字通常是不二人選，但是其中卻包含一些小陷阱 (嚴格來說不是陷阱，是實作 `IDisposable` 的時候沒做好)。

> 背景知識:  
> `using` 關鍵字就是 try-finally 加上呼叫 `IDisposable.Dispose()` 的語法糖，沒把握的話找個反組譯工具確認一下就知道了。

<!--more-->

### 有可能的問題
#### 捕捉不到的例外
首先，用下面的範例程式來說明：
``` csharp
public class Foo : IDisposable
{
    public void Dispose()
    {
        throw new Exception("a");
    }
}
```

``` csharp
// caller
using (var f = new Foo())
{
    try
    {
        // do something
    }
    catch (Exception)
    {
        // do something
    }
} // exception thrown here.

// never executed from here.
```

我們有一個實作 `IDisposable` 的類別，這個類別模擬在 `Dispose` 方法中拋出例外，而上面的程式碼因為 `Foo.Dispose()` 拋出例外而使得看起來能正常捕捉例外的程式碼其實是會有漏洞的。

#### 拋出預期外的例外
那如果把 try-catch 區塊移到 using 外呢？以下方程式來說明：  

``` csharp
public class Foo : IDisposable
{
    public void Dispose()
    {
        throw new Exception("a");
    }
}
```

``` csharp
// caller
try
{
    using (var f = new Foo())
    {
        throw new Exception("b");
    }
}
catch (Exception ex)
{
    // got exception from Foo.Dispose() and missing the one from try block.
}
```

一樣的 `Foo`，不一樣的呼叫端，但是這樣使用會造成呼叫端的 catch 區塊中捕捉到的例外其實是 `Foo.Dispose()` 拋出的例外，這意味著**當呼叫端程式發生例外時，錯誤根本不會被捕捉到**，也就代表當 Production Issue 發生時，會完全看不到呼叫端程式真正的例外，在有時間壓力下發生這種事是很可怕的。

#### 問題總結
這件事的根本原因是因為在 `using` 的 finally 區塊中拋出例外使得呼叫端誤以為自己有考慮到所有例外，或是呼叫端誤以為自己能捕捉到 try 區塊中的例外，但其實不然。

### 解決方案
#### 服務提供方確保 `IDisposable.Dispose()` 不拋出例外
如果要實作 `IDisposable`，必須確保 `IDisposable.Dispose()` 方法中不會拋出例外。

#### (不推薦) 呼叫端避免使用 `using`
就是用手動釋放資源取代 `using`，如下：

``` csharp
var f = new Foo();
try
{
    throw new Exception("a");
}
catch (Exception)
{
	// do something
}
finally
{
    // release in anther way instead of calling Dispose()
}
```

一般來說，這個問題應該是服務提供方應該要注意的，所以除非確定服務提供方有這個缺陷且沒辦法要求改正，不然不推薦將這種作法作為預設選項。

#### (很不推薦) 呼叫端用多個 try-catch 暴力解
很不推薦的做法，雖然簡單但太過暴力，很醜且維護的人很容易覺得這是多餘的而拆掉其中一個 try-catch 區塊。
``` csharp
try
{
    using (var f = new Foo())
    {
        try
        {
            throw new Exception("a");
        }
        catch (Exception ex)
        {
            // do something
        }
    }
}
catch (Exception ex)
{
    // do something else
}
```

### 結論
會想寫這篇是源於[這個已知的問題](https://learn.microsoft.com/en-us/dotnet/framework/wcf/samples/use-close-abort-release-wcf-client-resources)，但我覺得這是微軟的鍋，不應該因噎廢食而放棄 `using`，但萬一遇到了，還是要知道有這個現象來避免鬼打牆找不到問題，所以需要紀錄一下來加強印象。

### 參考
[Close and Abort release resources safely when network connections have dropped](https://learn.microsoft.com/en-us/dotnet/framework/wcf/samples/use-close-abort-release-wcf-client-resources)
