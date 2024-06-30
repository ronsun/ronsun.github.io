---
title: C# 語言特性更新 - C# 11
date: 2024-06-30 23:41:03
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->

### 特性可支援泛型 (Generic attributes)
可在特性上使用泛型參數。
``` csharp
public class GenericAttribute<T> : Attribute { }

[GenericAttribute<string>]
public string Method() => default;
```

但有一些限制：
1. 必須是已確定的型別，例如 `GenericAttribute<string>`；不能是不確定的泛型型別像是 `GenericAttribute<T>`。
2. 型別引數和 `typeof` 有相同的限制，不允許需要 metadata annotation 的型別，像是 `dynamic`、`string?` 等可為 null 的參考型別、`(int x, int y)` 等各種 Tuple。但上述情境分別可用 `object`、`string`、`ValueTuple<int, int>` 取代。

> 關於 metadata annotation 以及為什麼有這些限制應該找時間深入研究背後的機制。  

### 支援泛型數學運算 (Generic math support)
透過在介面中加入 `static abstract` 或 `static virtual` 的運算子多載來訂定一致的規範與慣例。且泛型數學運算支援下面的一些特性：
1. 運算子多載
    ``` csharp
    using System;

    public interface IMathOperations<TSelf>
        where TSelf : IMathOperations<TSelf>
    {
        static abstract TSelf operator +(TSelf left, TSelf right);
    }

    public struct MyNumber : IMathOperations<MyNumber>
    {
        public int Value { get; }

        public MyNumber(int value)
        {
            Value = value;
        }

        public static MyNumber operator +(MyNumber left, MyNumber right)
        {
            return new MyNumber(left.Value + right.Value);
        }

        public override string ToString() => Value.ToString();
    }

    public class Program
    {
        public static void Main()
        {
            MyNumber a = new MyNumber(10);
            MyNumber b = new MyNumber(5);

            MyNumber resultAdd = a + b;

            Console.WriteLine($"Addition: {resultAdd}"); // Output: Addition: 15
        }
    }
    ```

2. `checked` 運算子多載和隱含的 unchecked 運算子多載可以同時存在
    ``` csharp
    using System;

    public interface IMathOperations<TSelf> where TSelf : IMathOperations<TSelf>
    {
        static abstract TSelf operator +(TSelf left, TSelf right);
        static abstract TSelf operator checked +(TSelf left, TSelf right);
    }

    public struct MyNumber : IMathOperations<MyNumber>
    {
        public int Value { get; }

        public MyNumber(int value)
        {
            Value = value;
        }

        // Normal addition operator
        public static MyNumber operator +(MyNumber left, MyNumber right)
        {
            return new MyNumber(left.Value + right.Value);
        }

        // Checked addition operator
        public static MyNumber operator checked +(MyNumber left, MyNumber right)
        {
            return new MyNumber(checked(left.Value + right.Value));
        }

        public override string ToString() => Value.ToString();
    }

    public class Program
    {
        public static void Main()
        {
            MyNumber a = new MyNumber(int.MaxValue);
            MyNumber b = new MyNumber(1);

            // Using the generic method with checked context
            try
            {
                MyNumber resultChecked = checked(a + b);
                Console.WriteLine(resultChecked); // This will throw OverflowException
            }
            catch (OverflowException)
            {
                Console.WriteLine("Overflow in checked context");
            }

            // Using the generic method with normal addition
            MyNumber c = new MyNumber(10);
            MyNumber d = new MyNumber(5);
            MyNumber resultUnchecked = c + d;
            Console.WriteLine(resultUnchecked); // Output: 15
        }
    }
    ```

3. 不帶正負號的右移運算子 `>>>` (unsigned right-shift operator)
 `>>>` 是邏輯移位 (Logical Shift)，不同於 `>>` 是算數移位 (Arithmetic Shift)。

4. 寬鬆的位移運算子
第二個運算元可以不是 `int` 或隱含可轉換成 `int` 的型別。

> 上述特性有些在 C# 11 以前就已經存在，只是不是透過實作介面來達成。這個新特性比較偏向設計面，讓一些特性能經由介面來規範，但缺點是如果用舊版的方式實作也可以運作，維護上就很容易出現兩套做法。就我的角度而言，雖然有一點優點，但如果引入到沒有強力慣例約束的團隊中可能會造成更大的反效果。


### `IntPtr` 和 `UIntPtr` 的別名 (Numeric `IntPtr` and `UIntPtr`)
分別是 `nint` 和 `nuint`。

### 字串插補中的分行符號 (Newlines in string interpolations)
字串插補可直接斷行。

### List patterns
針對 List 和陣列的語法糖，像是這樣：
``` csharp
void Foo(IList<int> list)
{
    var ans = list switch
    {
        [1, 2, 3] => "Matched exactly [1, 2, 3]",
        [1, _, 3] => "Matched pattern [1, _, 3]",
        _ => "No match"
    };

    ans.Dump();
}
```

要讓 `switch` 模式比對可用， `list` 只能是陣列、 `IList<T>` 或其衍生型別。

### 改善的方法群組轉換至委派 (Improved method group conversion to delegate)
以下面範例比對差異：
``` csharp
using System;

public class Program
{
    public static void Main()
    {
        Action<string> action1 = PrintMessage;
        Action<string> action2 = PrintMessage;

        // In C# 10 and earlier, these will be different instances. Output: False
        // In C# 11, these may be the same instance due to caching. Output: True
        Console.WriteLine(object.ReferenceEquals(action1, action2));

        action1("Hello from action1!"); // Output: Hello from action1!
        action2("Hello from action2!"); // Output: Hello from action2!
    }

    public static void PrintMessage(string message)
    {
        Console.WriteLine(message);
    }
}
```

### 原始字串常值 (Raw string literals)
以 `"""` 開始與結尾，可支援任意文字，包括空白字元、分行符號、內嵌引號和其他特殊字元。可搭配 `$$` 使用。

> 但搭配  `$$` 使用時，規則有一些細節上的不同，先知道就好之後再補坑。

### 自動預設結構 (Auto-default struct)
自動把未初始化的欄位設為該型別預設值，如下面程式碼在舊版無法編譯但在 C# 11 可以編譯，未初始化的屬性 `Y` 會是 `default(int)`。  

> 是針對欄位，但屬性其實是 backing field + getter + setter 的組合，所以也適用。

``` csharp
public struct Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x)
    {
        X = x;
    }
}
```

### 字串常數模式比對可套用到 `Span<char>` 和 `ReadOnlySpan<char>` 上
如下方範例，在 C# 11 上可以運作。  
``` csharp
void ProcessSpan(ReadOnlySpan<char> span)
{
    switch (span)
    {
        case "Hello":
            Console.WriteLine("Greeting detected.");
            break;
        case "World":
            Console.WriteLine("World detected.");
            break;
        default:
            Console.WriteLine("Unknown span.");
            break;
    }
}
```

### 擴充 `nameof` 的適用範圍 (Extended nameof scope)
`nameof()` 可運用在特性上作為參數傳入，在參數驗證情境下可方便的印出有問題的參數名稱。

``` csharp
public void MyMethod([Validation(nameof(param))]int param)
{
    Console.WriteLine("Method executed.");
}
```

### UTF-8 字串常值 (UTF-8 string literals)
類似於 `123m` 表示 decimal `123`，`"Hello, 世界! 🌍"u8` 表示以 UTF-8 編碼的字串，省去一些轉碼的操作。
``` csharp
class Program
{
    static void Main()
    {
        ReadOnlySpan<byte> utf8StringLiteral = "Hello, 世界! 🌍"u8;

        // 48 65 6C 6C 6F 2C 20 E4 B8 96 E7 95 8C 21 20 F0 9F 8C 8D 
        foreach (byte b in utf8StringLiteral)
        {
            Console.Write($"{b:X2} ");
        }
    }
}
```

### 必要成員 (Required members)
可將 `required` 修飾詞加到屬性和欄位上，強迫建構子和呼叫端初始化這些值。  

### `ref` 欄位和 `ref scoped` 變數

### `file` 存取修飾詞 (File local types)
不同於其他存取修飾詞關注在 Assembly 或類別， `file` 以檔案為基準將可見度限縮在單一檔案中。

### 結論
從這陣子從 C# 7 一路追上來，到後面發現基於 C# 7 後面的版本的新特性另外再做的延伸我已經看不太懂了，等看完 C# 12 後還是要再回頭從特性的角度全面了解。

### 參考
ChatGPT  

[What's new in C# 11](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11)