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

[GenericAttribute<string>()]
public string Method() => default;
```

但有一些限制：
1. 必須是已確定的型別，例如 `GenericAttribute<string>`；不能是不確定的泛型型別像是 `GenericAttribute<T>`。
2. 型別引數和 `typeof` 有相同的限制，不允許需要 metadata annotation 的型別，像是 `dynamic`、`string?` 等可為 null 的參考型別、`(int x, int y)` 等各種 Tuple。但上述情境分別可用 `object`、`string`、`ValueTuple<int, int>` 取代。

> 關於 metadata annotation 以及為什麼有這些限制應該找時間深入研究背後的機制。  

### 支援泛型數學運算 (Generic math support)
1. 透過在介面中加入 `static abstract` 或 `static virtual` 的運算子多載來訂定一致的規範與慣例。
    ``` csharp
    using System;

    public interface IAdditionOperators<TSelf, TOther, TResult>
    {
        static abstract TResult operator +(TSelf left, TOther right);
    }

    public struct MyNumber : IAdditionOperators<MyNumber, MyNumber, MyNumber>
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
    }
    ```

    > 舊版 C# 本來就支援運算子多載，這個新特性比較偏向設計面，讓型別的運算子多載能經由介面來規範，但缺點就是用舊版的方式實作也可以運作，代表維護上很容易出現兩套做法。就我的角度而言，雖然有一點優點，但如果引入到沒有強力慣例約束的團隊中可能會造成更大的反效果。

2. 自定義 `checked` 運算子
    ``` csharp
    using System;

    public struct MyNumber
    {
        public int Value { get; }

        public MyNumber(int value)
        {
            Value = value;
        }

        // Checked addition operator
        public static MyNumber operator checked +(MyNumber left, MyNumber right)
        {
            return new MyNumber(checked(left.Value + right.Value));
        }

        // Normal addition operator (implicitly unchecked)
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
            MyNumber a = new MyNumber(int.MaxValue);
            MyNumber b = new MyNumber(1);

            // Using checked context with user-defined checked operator
            try
            {
                MyNumber resultChecked = checked(a + b);
                Console.WriteLine(resultChecked); // This will throw OverflowException
            }
            catch (OverflowException)
            {
                Console.WriteLine("Overflow in checked context");
            }

            // Using normal (implicitly unchecked) operator
            MyNumber resultUnchecked = a + b;
            Console.WriteLine(resultUnchecked); // Output: -2147483648 (overflow wraps around)
        }
    }
    ```

    > 實測也是舊版舊有的功能，只是在運算子多載上加上 `checked` 來顯式區分出有沒有防溢位機制而已。
3. 


### 結論

### 參考
