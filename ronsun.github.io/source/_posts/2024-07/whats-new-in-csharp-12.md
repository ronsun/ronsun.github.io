---
title: C# 語言特性更新 - C# 12
date: 2024-07-08 23:02:16
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->

### 預設建構子 (Primary constructors)
可套用到所有類別與結構。其他建構子可以選擇呼叫預設建構子。

``` csharp
public class Person(string name, int age)
{
    public string Name { get; } = name;
    public int Age { get; } = age;

    // Calling the primary constructor using `this(name, age)` is required.
    public Person(string name) : this(name, 0)
    {
    }
}
```

### 集合運算式 (Collection expressions)
像 `List<string> b = ["one", "two", "three"];` 這樣的語法能套用到更多 "類集合" 型別，包含陣列、`Span<T>`、`ReadOnlySpan<T>`、和其他支援集合初始設定式的型別。  

另外能像是下面範例這樣透過散佈運算子 (spread element) 來組合集合：
``` csharp
int[] row0 = [1, 2, 3];
int[] row1 = [4, 5, 6];
int[] row2 = [7, 8, 9];
int[] single = [.. row0, .. row1, .. row2];
```

### `ref readonly` 參數 (`ref readonly` parameters)

### Default lambda parameters
``` csharp
// Default increment to 1
var IncrementBy = (int source, int increment = 1) => source + increment;

Console.WriteLine(IncrementBy(5)); // 6
Console.WriteLine(IncrementBy(5, 2)); // 7
```

### 設定任何類型的別名 (Alias any type)
可透過 `using` 關鍵字設定任何型別的別名。
``` csharp
using System;
using IntArray = int[];

class Program
{
    static void Main()
    {
        IntArray numbers = new int[] { 1, 2, 3, 4, 5 };
        Console.WriteLine("Array elements:");
        foreach (var number in numbers)
        {
            Console.WriteLine(number);
        }
    }
}
```

> 又是個需要依賴強力的程式碼品質管理機制的特性。

### 內嵌陣列 (Inline arrays)
Runtime 或其他套件開發人員用以提升效能的機制，一般人通常不會用。

### `ExperimentalAttribute` (Experimental attribute)
用 `System.Diagnostics.CodeAnalysis.ExperimentalAttribute` 來標註實驗性功能，存取時編譯器會出現 warning。

> 以前不得已只能用 `ObsoleteAttribute` 搭配描述來標記實驗性功能，現在有更適當工具可以用了。

### Interceptors
這是個實驗性功能，以後可能會被移除或大改。

### 參考
ChatGPT

[What's new in C# 12](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12)