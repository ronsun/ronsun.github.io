---
title: 將 Stopwatch 封裝成小工具
date: 2019-01-12 21:00:02
categories:
- C#
- .NET
tags:
---

Stopwatch 經常被用來觀察程式的運行時間, 在開發階段能協助我們抓出一些效能的問題, 不過他的基本使用方式不是很漂亮, 在程式碼中插入一堆 Stopwatch 會干擾開發或是不小心沒移除而 commit 上版控也不好.   

另一方面, 如果有需要用他來觀察 production 環境的運作效能的時候, 這些穿插在主要程式碼中的 Stopwatch 就更加干擾了, 所以找了兩個方法來將 Stopwatch 封裝成小工具.  

<!--more-->

### Action
這個方式是將主要邏輯 `action()` 以及紀錄運行時間的方法 `report(ticks)` 做為 Action 傳進工具中執行, 並在執行前後加上 Stopwatch 計算運行時間後呼叫紀錄運行時間的方法 `report(ticks)`.  

其中第二個參數 `report(ticks)` 會存在主要是假設呼叫端對於回傳執行時間的所做的處理不同, 如果覺得兩個 Action 參數過於複雜, 可以把第二個參數改成選擇性參數, 沒傳入的話就執行預設的行為就好, 或者目前系統沒這個需求的話, 也可以不需要第二個參數.  

範例如下:  

``` csharp
public class StopwatchReporter
{
    public static void Execute(Action action, Action<long> report)
    {
        var stopwatch = Stopwatch.StartNew();

        action();

        stopwatch.Stop();
        var excusionTicks = stopwatch.ElapsedTicks;

        report(excusionTicks);
    }
}
```

這樣封裝非常簡單易懂, 使用上也還算方便, 可以輕易的將主要邏輯與效能監控分開, 如下:  
``` csharp
StopwatchReporter.Execute(
    () =>
    {
        // Do anything here
    }, 
    ticks => 
    {
        // Logging excusion time or something else here
    });

```

### 實作 IDisposable
上一個方法唯一的小缺點就是還是需要將主要流程包進 Action 裡面, 雖然 C# 提供的語法糖已經大幅提高可讀性了, 不過還是有其他方法能讓呼叫端看起來更簡潔.  

``` csharp
public class StopwatchReporter : IDisposable
{
    private string _name;
    private Stopwatch _stopwatch;

    public StopwatchReporter(string name)
    {
        _name = name;
        _stopwatch = new Stopwatch();
        _stopwatch.Start();
    }

    public void Dispose()
    {
        _stopwatch.Stop();
        var excusionTime = _stopwatch.ElapsedMilliseconds;

        // logging here
        Debug.WriteLine($"{_name} excusion time: {excusionTime} ms");

        _name = null;
        _stopwatch = null;
    }
}
```

這種做法最大的優點就是呼叫端極致簡潔, 如下:  
``` csharp
using (new StopwatchReporter("something"))
{
    //Do anything here...
}
```

但其實這種做法缺點比較多, 一來是實作複雜度提高很多, 且在我[所參考的文章下面的回覆中](http://pietschsoft.com/post/2015/12/17/Code-Tip-Simpler-Performance-Timer-Logging-in-C)有提到, 為了這個便利性而讓程式碼在釋放資源用的方法中去操作資源, 下面節錄自該則回覆:

> You implement IDisposable when you are managing resources. Implementing it communicates to other developers that you're doing dealing with memory usage, and implies that you SHOULD call Dispose as soon as reasonably possible. Using it as syntactic sugar confuses that message.  
> 
> Patterns and interfaces communicate intent. Using a pattern because you like the way it reads while ignoring what the interface intends creates cluttered code.

其實我個人是滿認同這則回覆的, 雖然對於呼叫端來說並不需要知道實作細節, 但就設計的角度來看, 的確是在一個釋放資源的 `Dispose()` 方法中做了意想不到的行為, 如果是單純把 Stopwatch 停止並釋放倒是還好, 但是如果要紀錄時間就可能會依賴其他組件, 當複雜度一提高, 這個做法的缺點就會更明顯.  

### 結論
兩種做法要選的話, 我會選第一種用 Action 的作法, 比較合理好懂且呼叫方式也夠整潔.  

但總體來說, 目前為止對於特別封裝這麼一個工具我還是覺得有點雞肋, 只是說如果真的很必要在真實的 production 環境監控的話也是一種可行方案, 就先記起來備忘, 搞不好哪天真的會用上.    

### 參考
[Exact time measurement for performance testing](https://stackoverflow.com/a/969327/8223582)  
[Code Tip: Simpler Performance Timer Logging in C#](http://pietschsoft.com/post/2015/12/17/Code-Tip-Simpler-Performance-Timer-Logging-in-C)  

---
