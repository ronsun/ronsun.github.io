---
title: 正確重拋例外
date: 2018-01-31 12:37:52
categories:
- C#
---

重拋例外有很多種方式, 包含 `throw`, `throw ex`, 使用 inner exception 以及 `System.Runtime.ExceptionServices.ExceptionDispatchInfo`, 或是不要重拋例外.  

先總結選擇如下順序:  
1. 最好不要重拋例外
1. 重拋優先選 `System.Runtime.ExceptionServices.ExceptionDispatchInfo`
1. 沒有框架支援則用 Inner Exception
1. `throw` 應該沒什麼情境需要用到了
1. `throw ex` 是具破壞性的作法, 除非是要**刻意破壞堆疊追蹤**

這篇會整理這幾種方法的使用與優缺, 並且另外提到 `throw` 和 `throw ex` 兩種方法對於堆疊追蹤的負面影響.  

<!--more-->

### throw & throw ex
#### throw vs throw ex
這兩個最常見也很相似, 所以一起說, 使用 throw 或是 throw ex 都能重拋例外, 但使用 `trow` 時會保留較完整的堆疊追蹤(stack trace), 而 `throw ex` 會重置堆疊追蹤, 造成行數顯示在 `throw ex` 那一行。  
另外值得注意的是, 用 `throw` 來重拋例外其實也會漏掉一些堆疊追蹤, 這會在下面展示。

下面的範例程式中直接使用 `throw` 重拋例外, 並印出堆疊追蹤內容。

```csharp    
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Security.Cryptography;
using System.IO;

namespace ronsun.github.io.lab
{
    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                ThrowException();
            }
            catch(Exception ex)
            {
                Console.WriteLine("======= ThrowException() =============");
                Console.WriteLine(ex);
            }
            Console.ReadLine();
        }
        
        static void ThrowException()
        {
            try
            {
                ExceptionHere();
            }
            catch(Exception ex)
            {
                throw;
            }
        }

        static void ExceptionHere()
        {
            throw new Exception("This is exception message.");
        }
    }
}
```

運行後可以看到堆疊追蹤最後是停在 `ExceptionHere()` 裡面(41行), 也就是真正引發例外的地方  

> 這邊有另外一個要注意的地方, 堆疊追蹤的第二行是停在 `throw` 的地方(35行), 本例看不出大的影響是因為有下一層呼叫可以追蹤, 下一小節會用另一段程式碼展示這個問題. 

```
======= ThrowException() =============
System.Exception: This is exception message.
   at ronsun.github.io.lab.Program.ExceptionHere() in C:\Users\Ron\Desktop\MyProjects\ronsun.github.io\ronsun.github.io.lab\Program.cs:line 41
   at ronsun.github.io.lab.Program.ThrowException() in C:\Users\Ron\Desktop\MyProjects\ronsun.github.io\ronsun.github.io.lab\Program.cs:line 35
   at ronsun.github.io.lab.Program.Main(String[] args) in C:\Users\Ron\Desktop\MyProjects\ronsun.github.io\ronsun.github.io.lab\Program.cs:line 17
```

但如果在35行使用 `throw ex` 的話, 堆疊追蹤最後就會停在 `ThrowException()` 這裡
```
======= ThrowException() =============
System.Exception: This is exception message.
   at ronsun.github.io.lab.Program.ThrowException() in C:\Users\Ron\Desktop\MyProjects\ronsun.github.io\ronsun.github.io.lab\Program.cs:line 35
   at ronsun.github.io.lab.Program.Main(String[] args) in C:\Users\Ron\Desktop\MyProjects\ronsun.github.io\ronsun.github.io.lab\Program.cs:line 17
```

#### throw 也會影響堆疊追蹤的內容

以下面的程式碼片段為例, 這次不另外呼叫一個引發例外的方法, 而是直接拋出一個例外
``` csharp
static void Main(string[] args)
{
    try
    {
        ThrowException();
    }
    catch(Exception ex)
    {
        Console.WriteLine("======= ThrowException() =============");
        Console.WriteLine(ex);
    }
    Console.ReadLine();
}

static void ThrowException()
{
    try
    {
        throw new Exception("This is exception message.");
    }
    catch(Exception ex)
    {
        throw;
    }
}
```

而此時的堆疊追蹤最後其實是停在 `throw` 那一行, 也就是說如果例外是發生在 `ThrowException` 方法中而不是下一層的呼叫, 且 `try` 區塊中有很多程式碼的時候, 還是會有難以除錯的困擾.  

### Inner Exception
基於前面的說明, 我們知道重拋例外會破壞堆疊追蹤, 所以另外一種做法是在重拋前將原始的例外放進 Inner Exception 中, 如下片段:

``` csharp
try
{
    DoSomething();
}
catch (Exception ex)
{
    // handle exception, then...
    throw new Exception("outer", ex);
}
```

但這樣做的缺點就是其實是重新包裝了例外, 如果呼叫端沒有往下查看 Inner Exception 的時候就會看不到完整的細節, 即使呼叫端有存取 Inner Exception 也比較麻煩, 是屬於功能正常但不夠優雅的方式.  

### System.Runtime.ExceptionServices.ExceptionDispatchInfo
靠框架解決, 是目前知道的方法中最漂亮的

``` csharp
try
{
    DoSomething();
}
catch (Exception ex)
{
    // handle exception, then...
    ExceptionDispatchInfo.Capture(ex).Throw();
}
```

使用容易, 看輸出也沒什麼副作用, 唯一的限制就是要依賴框架.  

### 不要重拋例外
這個方法寫在寫在這裡有點奇怪, 但個人來說是非常不喜歡重拋例外的, 比較傾向把所有**不需要特殊處理**的例外都讓全域例外處理機制去處理, 例如: [Exception filters](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#exception-filters).  

### 結論
雖然理想上是不要重拋例外, 但如果真的不得已需要, 則使用 `System.Runtime.ExceptionServices.ExceptionDispatchInfo`, 萬一使用的框架不支援的話, 那至少要使用 Inner Exception 去處理.  

### 參考資料

[Is there a difference between “throw” and “throw ex”?
](https://stackoverflow.com/questions/730250/is-there-a-difference-between-throw-and-throw-ex)  

[debuggability problems associated with catch / rethrow](https://blogs.msdn.microsoft.com/jmstall/2007/02/07/catch-rethrow-and-debuggability/)  

[‘throw e;’ vs. ‘throw;’](https://blogs.msdn.microsoft.com/jmstall/2007/02/15/throw-e-vs-throw/)  

[How to rethrow exception correctly in .Net](https://berserkerdotnet.github.io/blog/rethrow-exception-correctly-in-dotnet/)