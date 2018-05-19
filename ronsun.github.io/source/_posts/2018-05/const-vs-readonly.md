---
title: const 和 readonly 特性與使用時機
date: 2018-05-19 22:57:22
categories:
- C#
- Language Spec

---

在C#中 `const` 和 `readonly` 都可以被當作常數來使用, 但兩者在特性上有許多的差異, 使用上也有一些需要注意的地方. 

<!--more-->

### const
#### 說明
`const` 又稱"編譯時期常數", 實際值在編譯期間就會被取代到使用常數的各個地方, 所以相對的限制比較多, 下面只列舉一部分 `const` 的重要特性, 完整特性可參照 C# 規格書(5.0版 章節10.4).
+ 常數被視為靜態成員, 呼叫方法是 `MyClass.MY_CONST`.
+ 常數宣告中的型別只能是 sbyte, byte, short, ushort, int, uint, long, ulong, char, float, double, decimal, bool, string, enum, 或 reference type.

> 很神奇的一點是, 常數的型別可以是 reference type, 但是只能賦值 null, 例如:  
> `public const MyClass MY_COSNT = null;`  
> [stackoverflow](https://stackoverflow.com/questions/27699906/why-are-we-allowed-to-use-const-with-reference-types-if-we-may-only-assign-null)上有人討論過這個特性的用法 (不過我目前大概還不會這樣用它)

#### 使用時機
常數的推薦用法與時機
+ 這個值是不能在執行時期變動的
+ 存取修飾子**不建議**用 `public`, 因為基於[這個情境](#偏好使用-readonly-而非-const), `const` 不適合跨組件引用.

### readonly
#### 說明
`readonly` 是用在唯讀欄位上, 限制這個欄位只能在建構子或靜態建構子中被修改, 作為常數使用時又稱為執行時期常數, 執行的時候再去參考變數取得真正的值.

#### 使用時機
唯讀欄位的推薦用法與使用時機
+ 初始化後就不能再被變動
+ 不適合或無法用 const 的時候

> 唯讀欄位的值可以在建構子中被改變也讓一些人覺得他不是那麼唯讀, 例如下面的程式碼:  
> ```
public static class C1
{
    public readonly static string str = "a";

    static C1()
    {
        str = "b";
        str = "c";
    }
}  
```
> 唯讀欄位`str`被賦值"a"之後又在建構子中被改變了兩次值.


### 偏好使用 readonly 而非 const
考慮一個情境, 現在有一個類別庫 `DefaultLib`, 裡面有一個類別 `DefaultClass` 如下:
``` csharp
    public class DefaultClass
    {
        public const int AGE = 18;
        public readonly string NAME = "John";
    }
```

另一個專案 `Execute` 參考這個類別庫, 並印出 `AGE` 和 `NAME` 如下:
``` csharp
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(DefaultClass.AGE);
            Console.WriteLine(new DefaultClass().NAME);

            Console.Read();
        }
    }
```

執行後沒問題, 印出了 18 和 John .

接下來, 改成 `AGE = 20`, `NAME = "Smith"` 後只編譯 `DefaultLib` 專案, 並將 `DefaultLib.dll` 覆蓋掉舊的 dll 後直接執行 `Execute.exe`.

**結果印出了 18 和 Smith**

這其實就是一開始說的 `const` 作為編譯時期常數, 實際值在編譯期間就會被取代到使用常數的地方, 但 `readonly` 是參考變數取得真正的值, 所以 `Execute` 這個專案編譯後印出 `AGE` 與 `NAME` 的那兩行, 其實是
``` csharp
    Console.WriteLine(18);
    Console.WriteLine(new DefaultClass().NAME);
```

於是, 不管參考的 `DefaultLib.dll` 怎麼變化, 只要 `Execute` 專案不重新編譯, 第一行印的永遠是 18 ,但使用 `readonly` 就不會有這個問題.  


### 結論
這篇主要的重點是**偏好使用 readonly 而非 const**而已, 這也不是什麼鮮為人知的秘密, 相關的文章google一下就很多了, 而且都寫得更好, 會特別寫一篇只是想整理一下學習成果, 還意外發現了 `const` 可以是 reference type 的神奇特性. 

### 參考
C#語言規格書  
[C# - const vs static readonly](https://blog.johnwu.cc/article/c-sharp-const-vs-static-readonly.html)  
[Why are we allowed to use const with reference types if we may only assign null to them?](https://stackoverflow.com/questions/27699906/why-are-we-allowed-to-use-const-with-reference-types-if-we-may-only-assign-null)  
