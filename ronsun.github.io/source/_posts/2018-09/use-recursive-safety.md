---
title: 安全的使用遞迴
date: 2018-09-15 00:57:51
categories:
- C#
- .NET

tags:
---

遞迴有一個很大的好處是能用非常簡短的程式碼達到相對複雜很多的功能, 一般來說可讀性高很多, 但也伴隨著一些問題, 例如不慎引發堆疊溢位(stack overflow), 這篇主要是要紀錄怎麼安全的使用遞迴.  

<!--more-->

### 堆疊溢位
.NET 中的 `StackOverflowException` 有兩種觸發情境, 一種是在程式碼中刻意拋出的, 像這樣: `throw new StackOverflowException()`, 這和一般的例外沒什麼差別, 只要有 catch 到就可以做後續處理, 但另一種真的因為堆疊溢位所引發的例外, 根據 [MSDN 的說明](https://msdn.microsoft.com/en-us/library/w6sxk224.aspx)是無法被 catch 的, 且經過實測發現問題發生時, **程序會直接被終止**.  
堆疊溢位這個問題通常是由無限遞迴所引發的, 如果在 production 環境發生這個錯誤, 造成服務直停擺, 那真的會很悲劇.  

> 其實就 MSDN 看來, 似乎有其他方式可以讓系統不要因為堆疊溢位而停擺, 但太深奧了看不懂, 另一方面來說, 會出現這個問題大多是因為 bug, 所以目前不打算深入去看這一塊.

### 防止堆疊溢位

#### 控制遞迴深度
目前看到最簡明的方法是利用一個參數去紀錄目前堆疊的深度, 並在超過設定的最高深度時終止遞迴(或做其他相應的處理).  
``` csharp
public void Foo(int depth = 0)
{
    if (depth > 9)
    {
        return;
    }

    // do something...

    Foo(++depth);

    // do something...
}
```

#### 做好單元測試
即使有控制遞迴深度, 但還是有可能會把控制深度的地方寫錯, 所以單元測試還是寫完整點才能多一層保障.  

### 結論
雖然無限遞迴引發的問題比較嚴重, 但只要做好控制與測試, 其實遞迴還是很好用的, 比較需要在意的地方反而是因為遞迴相對抽象一點, 如果遞迴寫得太複雜的話, 後人會比較難維護.  

### 參考
[MSDN: Troubleshooting Exceptions: System.StackOverflowException](https://msdn.microsoft.com/en-us/library/w6sxk224.aspx)  
[What's a good general way of catching a StackOverflow exception in C#?](https://stackoverflow.com/questions/3871797/whats-a-good-general-way-of-catching-a-stackoverflow-exception-in-c) - 最佳解下面的推文部分有提到看似有方法處理堆疊溢位, 但有點矯枉過正 (a bit overkill)  
