---
title: LINQ 的延遲執行與誤用風險
date: 2024-10-03 00:14:37
categories:
- C#
- .NET
tags:
---

LINQ (Language Integrated Query)  的延遲執行機制 (Deferred Execution)  是其一大特色，讓查詢表達式在實際迭代前都不會執行。然而，如果不加注意，可能會因誤用而引發難以察覺的效能問題。本文將探討 LINQ 的延遲執行機制及其潛在的誤用風險，並提供實際程式碼範例說明。

<!--more-->

### 介紹延遲執行 (Deferred Execution) 
LINQ 的延遲執行意指查詢表達式不會立即執行，只有在真正需要取出資料時才進行計算。

#### 快速判斷是否會真正取出資料
許多人知道延遲執行的概念，也能列出一些延遲執行的運算子 (Deferred Execution Operators)  和會立即執行的運算子 (Immediate Execution Operators)  。但很少人能說明「如何分辨延遲執行運算子與立即執行運算子」。以下提供一些快速分辨方式，讓我們在看到陌生的 LINQ 方法時，能大概猜出其執行方式。

##### 延遲執行運算子 (Deferred Execution Operators)
幾乎所有的延遲執行運算子都會回傳 `IEnumerable<T>` 或 `IOrderedEnumerable<TElement>`，基本概念就是「Enumerable」，包含 `foreach` 搭配 `yield` 並回傳 `IEnumerable<T>` 的方法。

> 之所以說「幾乎」，是因為我們可以透過擴充方法實作回傳這些型別的 LINQ 方法，但如果內部實作觸及立即執行運算，那麼單從方法簽章判斷可能會誤判。  
> 但擴充會回傳 `IEnumerable<T>` 或 `IOrderedEnumerable<TElement>` 的 LINQ 方法卻觸及立即執行運算，我認為這是設計上的錯誤。

##### 立即執行運算子 (Immediate Execution Operators)
與延遲執行運算子不同，立即執行運算子的判定相對容易。只要是回傳具體的值或集合等非「Enumerable」型別的方法都是立即執行運算子，包括透過 `foreach` 直接取得運算結果的方式。

##### 小結
上述兩種 LINQ 運算子的分類只是初步判定。實際上，還有串流 (Streaming) 或非串流 (Nonstreaming) 兩種方式，區分內部是可能提前返回還是必須走訪所有元素，詳見 [Classification of standard query operators by manner of execution](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries#classification-of-standard-query-operators-by-manner-of-execution)。

### 誤用情境
用於展示誤用情境，所有範例都會使用下面的類別與資料示範。
``` csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

var source = new List<Person>
{
    new Person { Name = "Alice", Age = 25 },
    new Person { Name = "Bob", Age = 32 },
    new Person { Name = "Charlie", Age = 45 },
    new Person { Name = "Diana", Age = 38 },
    new Person { Name = "Eve", Age = 29 },
    new Person { Name = "Frank", Age = 40 },
    new Person { Name = "Grace", Age = 54 },
    new Person { Name = "Henry", Age = 3 },
    new Person { Name = "Ivy", Age = 30 },
    new Person { Name = "Jack", Age = 27 },
    new Person { Name = "Karen", Age = 35 },
    new Person { Name = "Leo", Age = 48 },
    new Person { Name = "Mia", Age = 22 },
    new Person { Name = "Nina", Age = 37 },
    new Person { Name = "Oscar", Age = 55 },
    new Person { Name = "Paul", Age = 15 },
    new Person { Name = "Quincy", Age = 42 },
    new Person { Name = "Rita", Age = 36 },
    new Person { Name = "Steve", Age = 25 },
    new Person { Name = "Tina", Age = 41 }
};
```

#### 重複運算 - LINQ
以下面範例為例：
``` csharp
var enmerable = source.Where(r => r.Age >= 18) ;

var oldestAdult = enmerable.MaxBy(r => r.Age) ;
var youngestAdult = enmerable.MinBy(r => r.Age) ;
```

由於 `Where()` 是延遲執行運算子，因此在執行 `MaxBy()` 和 `MinBy()` 時，`source` 會被篩選兩次，造成不必要的運算浪費。改進方式如下：

``` csharp
var adults = source.Where(r => r.Age >= 18) .ToList() ;

var oldestAdult = adults.MaxBy(r => r.Age) ;
var youngestAdult = adults.MinBy(r => r.Age) ;
```

透過先使用 `ToList()` 方法將符合條件的成人資料篩選出來，後續的 `MaxBy()` 和 `MinBy()` 就不會重複篩選。  

#### 重複運算 - `foreach`
以下面範例為例：
``` csharp
var enmerable = source.Where(r => r.Age >= 18) ;

foreach (var item in enmerable) 
{
    // Do something.
}
foreach (var item in enmerable) 
{
    // Do something else.
}
```

類似於上一個情境，不同之處在於這裡由 `foreach` 觸發。修正方式同樣是提前呼叫立即執行運算，避免後續的兩個 `foreach` 造成重複的` Where()` 運算。

#### 重複運算 - 子查詢
子查詢是容易被忽略的情境，以下範例：
``` csharp
var enumerable = source.Where(r => r.Age >= 18) ;
var targetAges = new List<int> { 40, 54, 23 };
var result = targetAges
    .Where(age =>
        enumerable.Any(s => s.Age == age) 
        && enumerable.Any(s => (s.Age / 10)  > 3) ) 
    .ToList() ;
```

由於兩次立即執行運算 `Any()` 被包含在子查詢中，容易忽略重複運算。解決方式如下：

``` csharp
var list = source.Where(r => r.Age >= 18) .ToList() ;
var targetAges = new List<int> { 40, 54, 23 };
var result = targetAges
    .Where(age =>
        list.Any(s => s.Age == age) 
        && list.Any(s => (s.Age / 10)  > 3) ) 
    .ToList() ;
```

#### 重複運算 - 不洽當的客製方法
參考以下客製方法：
``` csharp
public static IEnumerable<T> Duplicate<T>(this IEnumerable<T> source) 
{
    foreach (var item in source) 
    {
        yield return item;
    }

    foreach (var item in source) 
    {
        yield return item;
    }
}
```

呼叫端如下：
``` csharp
var enmerable = src
    .Where(r => r.Age >= 18) 
    .Duplicate() 
    .ToList() ;
```

問題在於 `Duplicate()` 方法中包含了兩個 `foreach`。即使搭配了 `yield return` 使其為延遲執行運算，但當呼叫端執行時，`Where()` 方法會在 `Duplicate()` 中執行兩次。

以這個範例來說，根本原因是 `Duplicate() ` 方法的設計錯誤，應該改寫這個方法來解決問題，但如果無法修改 `Duplicate()` 方法，臨時解決方式如下：

``` csharp
var enmerable = src
    .Where(r => r.Age >= 18) 
    // Workaround if we have no chance to correct Duplicate() 
    .ToList() 
    .Duplicate() 
    .ToList() ;
```

#### 重複運算 - 搭配外層迴圈或迭代器
有時候我們會想要利用 `Any()` 的效果減少不必要的重複運算，但如果沒有仔細分析可能會適得其反，以下面程式碼為例：  

``` csharp
IEnumerable<int> GetNumbers()
{
    List<int> numbers = [1, 2, 3, 5, 6, 7, 8, 9]; // GetNumbersFromDatabase()
    Console.WriteLine("GetNumbersFromDatabase() executed");

    for (int i = 0; i < numbers.Count; i++)
    {
        Console.WriteLine($"Yielding {numbers[i]}");
        yield return numbers[i];
    }
}

var outer = new List<int> { 1, 2, 3 };
var numbers = GetNumbers();
//var numbers = GetNumbers().ToList();
foreach (var n in outer)
{
    if (numbers.Any(r => r % 2 == 0))
    {
        Console.WriteLine($"An even found");
    }
}
```

首先，我們從 Case 1 可以看到試圖透過不先執行 `ToList()` 找出所有號碼，而是在 `Any()` 被呼叫時才去迭代，減少 Yielding 這邊的執行次數與成本，但卻重複執行了 `GetNumbersFromDatabase() executed` 的部分，當 `GetNumbersFromDatabase()` 成本遠高於 Yielding 的時候，整體的效能其實是更差的，輸入如下：
```
GetNumbersFromDatabase() executed
Yielding 1
Yielding 2
An even found
GetNumbersFromDatabase() executed
Yielding 1
Yielding 2
An even found
GetNumbersFromDatabase() executed
Yielding 1
Yielding 2
An even found
```

反之，Case 2 的部分，雖然看似執行了所有的 Yielding，但因為提早執行完 `GetNumbers()` 並得到所有結果，反而省下了 `GetNumbersFromDatabase()` 的高成本，輸出如下：
```
GetNumbersFromDatabase() executed
Yielding 1
Yielding 2
Yielding 3
Yielding 4
Yielding 5
Yielding 6
Yielding 7
Yielding 8
Yielding 9
An even found
An even found
An even found
```

甚至我們可以更進一步思考，即使沒有 `GetNumbersFromDatabase()` 的消耗，只要外層 `outer` 的數量夠多，Case 2 的效能就可能會比 Case 1 好。


#### 小結
其實誤用情境主要是重複迭代。但由於 LINQ 方法的組合與類似子查詢的語法結構，很多時候我們未察覺其中的重複迭代，進而無意間犯錯。

### 結論
理解 LINQ 的延遲執行機制有助於我們撰寫更高效的程式碼。透過注意可能的重複迭代情境，並適時使用立即執行運算子，可以避免不必要的效能損耗。

### 參考
[Classification of standard query operators by manner of execution](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries#classification-of-standard-query-operators-by-manner-of-execution)   

ChatGPT