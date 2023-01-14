---
title: 關於物件和集合初始設定式
date: 2022-06-06 01:17:59
categories:
- C#
- Language Spec
tags:
---

寫 C# 也好幾年了, 對於物件和集合初始設定式的使用也很習慣, 也知道那是語法糖, 但是前陣子看到自訂類別要套用物件和集合初始設定式時, 才發現其實沒有想像中的熟, 所以想稍微再整理一下一些細節.  

<!--more-->

首先, 其實[官方文件](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers)已經非常完整的, 如果看過官方文件其實這邊就沒什麼好看的了.

也因為官方文件很完整, 所以這邊只會記一些平常沒被注意到的細節.  

### 初始設定式的語法糖
#### Indexer 部分
以 `Dictionary` 為例
``` csharp
var x = new Dictionary<string, string>
{
    ["k"] = "v",
}
```

編譯後的結果會接近下面的樣子, 去存取 Indexer.
``` csharp
var x = new Dictionary<string, string>();
x["k"] = "v";
```

#### Collection 部分 
以 List 為例
``` csharp
var x = new List<string>
{
   "a",
   "b",
}
```

編譯後的結果會接近下面的樣子, 去存取 `Add()` 方法.
``` csharp
var x = new List<string>();
x.Add("a");
x.Add("b");
```

### 自定義類別套用初始設定式
#### Indexer 部分
如標題, **重點就是要實作對應的 indexer**.  

懶得自己想 Demo 了, 就偷一下官方的範例來改:  
``` csharp
// 自定義的物件與 indexer
public class Matrix
{
    private double[,] storage = new double[3, 3];

    public double this[int row, int column]
    {
        get => storage[row, column];
        set => storage[row, column] = value;
    }
}
```

``` csharp
// 使用
var identity = new Matrix
{
    [0, 0] = 1.0,
    [0, 1] = 0.0,
    [0, 2] = 0.0,

    [1, 0] = 0.0,
    [1, 1] = 1.0,
    [1, 2] = 0.0,

    [2, 0] = 0.0,
    [2, 1] = 0.0,
    [2, 2] = 1.0,
};

// identity[0, 0] => 1
// identity[0, 1] => 0
```

#### Collection 部分 
兩個重點
+ **必須實作 `System.Collections.IEnumerable` (或其衍生類別/介面)**  
  - 這部分倒也不是初始設定式會用到, 只是不實作的話呼叫時就無法使用集合初始設定式.  
  - `GetEnumerator()` 內容在初始設定式不會被呼叫到, 所以沒實作也不會壞, 但是還是應該要實作.
+ **必須有 Add 方法**
  - Add 方法參數對應的就是初始設定式中參數的數量, 例如 `{a, b, c}` 就是對應呼叫到 `Add(a, b, c)` 這個多載

``` csharp
// 自定義的物件
public class NameCollection : IEnumerable
{
    private IList<string> _names = new List<string>();

    public void Add(string lastName)
    {
        _names.Add($"{lastName}");
    }

    public void Add(string fistName, string lastName)
    {
        _names.Add($"{fistName} {lastName}");
    }

    public IEnumerator GetEnumerator()
    {
        return _names.GetEnumerator();
    }
}
```

``` csharp
// 使用
var fullNames = new NameCollection
{
	// Add("Smith")
    {"Smith"},
	// Add("John", "Smith")
    {"John", "Smith"},
};

// 這邊會需要用到實作的 IEnumerable.GetEnumerator()
foreach(var f in fullNames)
{
    var fullName = f.ToString();
}
```

### 結論
其實使用上都很熟了, 但碰到自定義類別要套用初始設定式時, 一時之間會想不到怎麼做 (尤其時集合初始設定式), 所以這邊紀錄一下怎麼實作.  

### 參考
[Object and Collection Initializers](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers)  

[Collection initializers in custom classes](https://riptutorial.com/csharp/example/160/collection-initializers-in-custom-classes)  

[Collection Initializers with Parameter Arrays](https://riptutorial.com/csharp/example/6054/collection-initializers-with-parameter-arrays)  
