---
title: 責任鏈模式 - Fluent 風格
date: 2024-04-04 01:03:34
categories:
- DesignPatterns
tags:
---

一般範例中的責任鏈，負責組合各實作的 Handler 時通常會如下方這樣組合，這樣的做法不管從操作上還是閱讀上都比較不友善：
```csharp
// Init
IHandler positiveEven = new PositiveEvenHandler();
IHandler negativeEven = new NegativeEvenHandler();
IHandler positiveOdd = new PositiveOddHandler();
IHandler negativeOdd = new NegativeOddHandler();

// Configure
positiveEven.SetNext(negativeEven);
negativeEven.SetNext(positiveOdd);
positiveOdd.SetNext(negativeOdd);

// Call the chain
positiveEven.Handle(1);
positiveEven.Handle(-1);
positiveEven.Handle(2);
positiveEven.Handle(-2);
```

這篇主要是紀錄透過簡單的修改讓 Handler 能更簡單的組合責任鏈。

<!--more-->

### 以 Fluent API 風格組合
根據上面的範例，如果能將組合責任鏈改成 Fluent API 風格，能有效提升易用性，如下：
``` csharp
// Init and Configure
IHandler root = new PositiveEvenHandler();
root.SetNext(new NegativeEvenHandler())
    .SetNext(new PositiveOddHandler())
    .SetNext(new NegativeOddHandler());

// Call the chain
root.Handle(1);
root.Handle(-1);
root.Handle(2);
root.Handle(-2);
```

而各節點的實作重點則是將 `void SetNext(IHandler next)` 改成回傳下一個節點的實例 `IHandler SetNext(IHandler next)`，非常簡單就能讓整個責任鏈更易用，如下範例：
``` csharp
public interface IHandler
{
    IHandler SetNext(IHandler handler);

    void Handle(int number);
}

public class PositiveEvenHandler : IHandler
{
    private IHandler _nextHandler;

    public IHandler SetNext(IHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public void Handle(int number)
    {
        if (number > 0 && number % 2 == 0)
        {
            Console.WriteLine($"Positive even number {number}");
            return;
        }

        if (_nextHandler != null)
        {
            _nextHandler.Handle(number);
        }
    }
}

public class NegativeEvenHandler : IHandler
{
    private IHandler _nextHandler;

    public IHandler SetNext(IHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public void Handle(int number)
    {
        if (number < 0 && -number % 2 == 0)
        {
            Console.WriteLine($"Negative even number {number}");
            return;
        }
        if (_nextHandler != null)
        {
            _nextHandler.Handle(number);
        }
    }
}

public class PositiveOddHandler : IHandler
{
    private IHandler _nextHandler;

    public IHandler SetNext(IHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public void Handle(int number)
    {
        if (number > 0 && number % 2 == 1)
        {
            Console.WriteLine($"Positive odd number {number}");
            return;
        }

        if (_nextHandler != null)
        {
            _nextHandler.Handle(number);
        }
    }
}

public class NegativeOddHandler : IHandler
{
    private IHandler _nextHandler;

    public IHandler SetNext(IHandler handler)
    {
        _nextHandler = handler;
        return handler;
    }

    public void Handle(int number)
    {
        if (number < 0 && -number % 2 == 1)
        {
            Console.WriteLine($"Negative odd number {number}");
            return;
        }

        if (_nextHandler != null)
        {
            _nextHandler.Handle(number);
        }
    }
}
```

### 其他作法 - Next 屬性取代 SetNext 方法
也是個簡化組合的程式碼的方式，但這種方式當整個責任鏈很長時可讀性稍差，搭配 DI 操作起來也不太順暢。 但優點就是直接用自動實作屬性就好，不用再額外實作 SetNext 方法。

``` csharp
IHandler chain = new PositiveEvenHandler
{
    Next = new 
    {
        Next = new NegativeEvenHandler
        {
            Next = new PositiveOddHandler
            {
                Next = new NegativeOddHandler{}
            }
        }
    }
};
```

### 結論
之前的文章提到過的，設計模式只是個樣板，實務上應該針對語言特性與應用場景做適當的調整。

### 參考
[Design Pattern— 責任鏈模式(Chain of Responsibility Pattern)](https://ad57475747.medium.com/design-pattern-%E8%B2%AC%E4%BB%BB%E9%8F%88%E6%A8%A1%E5%BC%8F-chain-of-responsibility-pattern-29757935134e)  