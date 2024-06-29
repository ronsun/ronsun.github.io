---
title: C# 語言特性更新 - C# 10
date: 2024-06-29 22:37:12
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->

### 實值型別的 Record (Record structs)
用 `record struct` 或是 `readonly record struct` 來定義實值型別的 Record。  

### `struct` 型別的改善
可以在結構類型上加上無參數建構子並在其中初始化欄位或屬性。且 `with` 左邊的型別可以是結構型別或是匿名參考型別。

### 插補字串處理常式 (Interpolated string handler)
沒用過，但從 ChatGPT 的說明看起來，像是可以把字串插補轉換成特定的 handler 並輸出客製化結果而不是預設的字串。但值得注意他背後的機制不像是轉型，而是編譯器在特定情境(以字串插補為引數傳入時)才會將其編譯成呼叫 `CustomInterpolatedStringHandler`。  

以 ChatGPT 的範例展示如下：  

``` csharp
using System;
using System.Runtime.CompilerServices;
using System.Text;

[InterpolatedStringHandler]
public ref struct CustomInterpolatedStringHandler
{
    private StringBuilder _builder;

    public CustomInterpolatedStringHandler(int literalLength, int formattedCount)
    {
        _builder = new StringBuilder(literalLength);
        Console.WriteLine($"Literal length: {literalLength}, formatted count: {formattedCount}");
    }

    public void AppendLiteral(string literal)
    {
        _builder.Append(literal);
        Console.WriteLine($"Appended literal: {literal}");
    }

    public void AppendFormatted<T>(T value)
    {
        _builder.Append(value?.ToString());
        Console.WriteLine($"Appended formatted: {value}");
    }

    public void AppendFormatted<T>(T value, string format) where T : IFormattable
    {
        _builder.Append(value?.ToString(format, null));
        Console.WriteLine($"Appended formatted with format: {value} with format {format}");
    }

    public void AppendFormatted(string value)
    {
        _builder.Append(value);
        Console.WriteLine($"Appended formatted string: {value}");
    }

    public void AppendFormatted(string value, int alignment)
    {
        _builder.Append(value.PadRight(alignment));
        Console.WriteLine($"Appended formatted string with alignment: {value} with alignment {alignment}");
    }

    public void AppendFormatted<T>(T value, int alignment) where T : IFormattable
    {
        _builder.Append(value?.ToString()?.PadRight(alignment));
        Console.WriteLine($"Appended formatted with alignment: {value} with alignment {alignment}");
    }

    public void AppendFormatted<T>(T value, int alignment, string format) where T : IFormattable
    {
        _builder.Append(value?.ToString(format, null)?.PadRight(alignment));
        Console.WriteLine($"Appended formatted with alignment and format: {value} with alignment {alignment} and format {format}");
    }

    public override string ToString() => _builder.ToString();
}

public static class CustomLogger
{
    public static void Log(CustomInterpolatedStringHandler handler)
    {
        Console.WriteLine(handler.ToString());
    }
}

class Program
{
    static void Main()
    {
        CustomLogger.Log($"Hello, {Environment.UserName}. Today is {DateTime.Now:MMMM dd, yyyy}.");
    }
}

```

輸出是像這樣：
```
Literal length: 7, formatted count: 2
Appended literal: Hello, 
Appended formatted: Ron
Appended literal: . Today is 
Appended formatted with format: 29 June, 2024 with format MMMM dd, yyyy
Hello, Ron. Today is 29 June, 2024.
```

> `CustomInterpolatedStringHandler` 類似於 duck typing 的概念，依賴於實現特定的方法，但我個人不太喜歡這種風格，因為沒有實作介面或繼承父類別時所能得到的相關方法的提示，只能靠記憶或查文件去寫，維護的時候也會多出很多需要記憶的細節。  
> 另外這個轉換行為太隱晦，看來是需要搭配完整的文件說明才能使用的特性，不然維護的人會很難意識到有客製化的字串插補。  

### 全域 `using` 指示詞 (Global using directives)
用 `global using` 語法來讓整個專案預設引入常用命名空間，減少單一個 `*.cs` 檔案中的 `using`。  

> 如果有使用夠完善的 IDE 下看起來弊大於利，方便沒多少但容易衍生更多的維護成本，全域的命名空間和各檔案間命名空間下同名類別混淆的情況會更難排查，要避免的話全域的 `using` 還得經過嚴謹的設計、規範與程式碼審查制度，會增加不少維護成本。  

### 檔案範圍的命名空間宣告 (File-scoped namespace declaration)
就省掉一個縮排而已。

``` csharp
// Just save 1 indention
namespace MyNamespace;

class A
{
    // ....
}


class B
{
    // ....
}
```

> 初步看來是個幫助很小的功能，預設有套用就用，沒套用的話也沒必要刻意去改。

### 擴充屬性在模式比對時的模式 (Extended property patterns)  
``` csharp
// before
if (person is { Address: { City: "Springfield" } })
{
    //...
}
// after
if (person is { Address.City: "Springfield" } })
{
    //...
}
```

### Lambda 運算式改善 (Lambda expression improvements)
1. 編譯器可以從 Lambda 運算式或方法群組推斷委派類型，例如：`var add = (int x, int y) => x + y;` 可以使用推斷型別 `var`。
2. 如果無法推論型別則要自己定義。
3. Attribute 可以加在 Lambda 運算式上。  

這讓 Lambda 運算式更像一個方法。

### 常數差補字串 (Constant interpolated strings)
如果字串插補的元素都是常數，則字串插補的結果可為常數，在編譯期確定字串內容來提升效能和安全性。  

``` csharp
const string firstName = "John";
const string lastName = "Doe";
// Both firstName and lastName are constant, so fullName can be constant.
const string fullName = $"{firstName} {lastName}";
```

### Record 的 `ToString` 方法可禁止繼承 (Record types can seal ToString)  

``` csharp
using System;

public record Person(string FirstName, string LastName)
{
    // Sealed ToString()
    public sealed override string ToString()
    {
        return $"{FirstName} {LastName}";
    }
}
```

### Assignment and declaration in same deconstruction
就可以更靈活的使用解構子，如下
``` csharp
// before 1
(int x, int y) = point;

// before 2
int x1 = 0;
int y1 = 0;
(x1, y1) = point;

// after
int x = 0;
(x, int y) = point;
```

### 改善明確指派 (Improved definite assignment)  
此改進主要是針對編譯器在分析變數是否已被正確初始化時提供更準確的判斷，減少誤判的空值警告。

### 在方法上允許 AsyncMethodBuilder 特性 (Allow AsyncMethodBuilder attribute on methods)  
允許在方法上加 `AsyncMethodBuilderAttribute`。


### CallerArgumentExpression attribute diagnostics

``` csharp
using System;
using System.Runtime.CompilerServices;

public static class Validator
{
    public static void Ensure(bool condition, 
        [CallerArgumentExpression(nameof(condition))] string? message = null)
    {
        if (!condition)
        {
            throw new ArgumentException($"Condition failed: {message}");
        }
    }
}

class Program
{
    static void Main(string[] args)
    {
        int x = 5;
        int y = 10;

        // This will not throw an exception
        Validator.Ensure(x < y);

        // This will throw an exception with a detailed message
        // "Condition failed: x > y"
        Validator.Ensure(x > y);
    }
}
```
如上範例，讓 `codition` 的參數內容 (x > y) 做為 `message` 的值，方便除錯。  

### 結論
C# 10 大多是一些比較小的擴展，看起來輕鬆多了。

### 參考
[What's new in C# 10](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10)  

ChatGPT