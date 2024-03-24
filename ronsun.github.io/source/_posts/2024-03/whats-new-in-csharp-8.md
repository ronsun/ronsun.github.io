---
title: C# 語言特性更新 - C# 8
date: 2024-03-24 22:38:17
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->

### 唯讀成員 (Readonly members)
允許在結構 (struct) 成員上單獨使用 `readonly` 修飾詞。  
其中有一些精細的限制，違反的話會有 Warning 或因而無法編譯，所以不用刻意去記。

### 預設介面方法 (Default interface methods)
介面中可以有預設實作方法，打破以前介面無法實作的限制。  
主要的優點是讓 API 開發者可以在後續版本上加上新的介面方法而不會造成破壞性變更 (Breakin Changes)，算是方便但是預設介面方法有很多限制，並不是表面上的 "介面中的方法可以有實作內容" 這麼單純，這個之後再另外發文探討。而這個特性除了方便外，也會讓介面和抽象方法之間的界線更為模糊，這時候設計的時候就應該更從物件導向設計的角度來決定兩者的使用時機，而不單單只是看語言特性的差異。  


### 擴大模式比對的應用 (More patterns in more places)
模式比對從 C# 8 後更新的非常迅速，五花八門琳瑯滿目的很多，如果不是要看變更歷程的話，推薦直接看完整教學：
+ [Pattern matching overview](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)  
+ [Patterns](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns)

#### `switch` 運算式 (Switch expressions)
讓 `switch` 語句更精簡，如下範例：
``` csharp
public enum Color
{
    Red,
    Green,
    Blue,
    Yellow
}

public string GetColorHex(Color color)
{
    return color switch
    {
        Color.Red => "#FF0000",
        Color.Green => "#00FF00",
        Color.Blue => "#0000FF",
        Color.Yellow => "#FFFF00",
		// Default
        _ => throw new ArgumentOutOfRangeException(nameof(color), color, null)
    };
}
```

這不只是語法外觀的改變，switch 區塊的主體也變成了運算式 (Expression)，而不是以前的陳述式 (Statement)。

#### 屬性模式 (Property patterns)、Tuple 模式 (Tuple patterns)、位置模式 (Positional patterns)
雖然命名上是 Property patterns，但他的意思不是在 Property 上使用模式比對，而是使用模式比對時可以針對目標型別的屬性來比對。大概就是用一堆符號來表達對屬性的型別、值範圍等判斷，主訴是程式碼的簡潔。  

Tuple 模式和位置模式依此類推。

### `using` 宣告 (Using declarations)
不同於以往的
``` csharp
using (var foo = new Foo())
{
    // Somthing to do
}
```

而改用
``` csharp
using var foo = new Foo();
// Somthing to do
```

差別在於以往我們要將 `using` 區塊另外框起來，常常需要處理區塊內宣告的參數在區塊外無法使用的問題，不難但很雜，且經常被迫要把宣告與賦值分開，程式碼就比較亂，差異如下：
``` csharp
var v1;
using (var foo = new Foo())
{
    v1 = foo.GetV1();
}

var v2 = v1 * 2;
```

``` csharp
using var foo = new Foo();
// Place only code related to "foo" in this scope to make it disposed ASAP.
var v1 = foo.GetV1();
var v2 = v1 * 2;
```

如上範例註解所示，在新的語法中我們不用關心區塊，區塊範圍會在編譯時期決定，但相對的就是要注意 `using` 宣告的變數應該盡早使用，避免在中間放其他無關程式碼造成實際的 `using` 區塊太大。


### 靜態區域函式 (Static local functions)
靜態區域函式能讓我們的區域函式不捕捉外層的變數，但這部分是透過 Warning `CS8421` 來達到的。  

區域函式的變數捕捉特性會產生關注點發散到外層方法的副作用，又因為預設會自動捕捉外層的變數導致平常開發時很難確保所需資訊都由參數列提供而不是透過變數捕捉，靜態區域函式的特點可以有效避免這個困擾。

### `ref struct` 可實作 `IDisposable` (Disposable ref structs)
就字面上的意思，同時也適用 `readonly ref struct`。

### 可為 Null 的參考型別 (Nullable reference types)
參考型別本來就可以是 Null，這個特性使得參考型別編譯時被視為不可為 Null，並由編譯器分析是否可能誤用使其值為 `null`。  
看起來是在空值判斷這個議題上，透過強制規範迫使開發者嚴格檢視空值情境，如果在已有嚴格規範的專案上使用有錦上添花的效果，但如果現有專案規範與品質不夠或開發習慣不嚴謹，啟用這個特性幫助不大，甚至適得其反讓程式碼風格更不一致。  

### 非同步資料流 (Asynchronous streams)
雖然名字有 streams 字眼，但他和 `Stream` 型別無關，是提供非同步迭代的一種特性。 

如下方範例程式，有幾個關鍵：
1. `async` 方法
2. 回傳 `IAsyncEnumerable<T>`
3. 呼叫端迭代時需要在前面加 `await` 關鍵字，不限定使用 `foreach` 的情境，在自己操作迭代器的情境也適用如 `await using var enumerator = numberGenerator.GetAsyncEnumerator();`。

``` csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        await foreach (var number in GenerateNumbersAsync(5))
        {
            Console.WriteLine(number);
        }
    }

    static async IAsyncEnumerable<int> GenerateNumbersAsync(int count)
    {
        for (int i = 0; i < count; i++)
        {
            await Task.Delay(1000);
            yield return i;
        }
    }
}
```

上面範例每秒都能回傳一個結果，但以前通常只能等到 `GenerateNumbersAsync()` 完成再一次回傳如下：
``` csharp
static async Task<IEnumerable<int>> GenerateNumbersAsync(int count)
{
    var numbers = new List<int>();
    for (int i = 0; i < count; i++)
    {
        await Task.Delay(1000);
        numbers.Add(i);
    }
    return numbers;
}

// Doesn't work, returns `IEnumerable<Task<int>>` also doesn't work.
// static async Task<IEnumerable<int>> GenerateNumbersAsync(int count)
// {
//     for (int i = 0; i < count; i++)
//     {
//         await Task.Delay(1000);
//         yield return i;
//     }
// }
```

而根據 ChatGPT 的範例，不使用這個特性要每秒都能回傳一個結果的話，程式碼就會像下面這樣難以閱讀：
``` csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        await GenerateNumbersAsync(5, async number =>
        {
            Console.WriteLine(number);
            await Task.Delay(1000);
        });
    }

    static async Task GenerateNumbersAsync(int count, Func<int, Task> processNumber)
    {
        for (int i = 0; i < count; i++)
        {
            await processNumber(i);
        }
    }
}
```

### `IAsyncDisposable` (Asynchronous disposable)
如字面上的非同步 dispose，上面的非同步資料流中提到的 `IAsyncEnumerable<T>` 就繼承自 `IAsyncDisposable`。

### 索引和範圍 (Indices and ranges)
用更簡潔的語法表達索引和範圍，提供兩個新的型別 `System.Index` 和 `System.Range`，並透過 `^` (倒數) 和 `..` (到) 符號來使用。 要注意的是第一個是 `0`，但倒數第一個是 `^1` 。  

歸納一下規則：
1. `^n` 是倒數第 n 個 (即 `sequence[sequence.Length - n]` 的概念)。
1. `x..y` 是索引 x (包含) 到索引 y (不包含)，省略 x 代表從頭開始。省略 y 代表到底為止。 x 和 y 可以是 ^m 和 ^n 這樣表示。 索引 x 必須小於或等於索引 y，而 ^m 和 ^n 推論成 x 和 y 後亦同。

範例如下：
``` csharp
string str = "This is a book.";
Index first = 0;
Index firstFromEnd = ^1;
str[first].Dump(); // T
str[firstFromEnd].Dump(); // .
str[0].Dump(); // T
str[^1].Dump(); // .

Range range = 10..14;
Range rangeFromEnd = ^5..^1;
str[range].Dump(); // book
str[rangeFromEnd].Dump(); // book
str[10..14].Dump(); // book
str[^5..^1].Dump(); // book

str[5..^8].Dump(); // is
str[^10..7].Dump(); // is

str[..7].Dump(); // This is
str[10..].Dump(); // book.
str[0..^0].Dump(); // This is a book.
str[..].Dump(); // This is a book.
```

雖然正向和倒數可以混用，但不要濫用不然可讀性也會差。

### `??=` (Null-coalescing assignment)
範例：
``` csharp
message ??= "Default Message";
```

同義於
``` csharp
if (message == null)
{
    message = "Default Message";
}
```

### 非託管結構型別 (Unmanaged constructed types)
如果結構只包含非託管型別的屬性時，該結構也是非託管型別。  

### 巢狀運算式中的 Stackalloc (Stackalloc in nested expressions)
如果 stackalloc 運算式的結果是 `Span<T>` 或 `ReadOnlySpan<T>` 就可以不用另外指派給另外一個變數，如下範例：
``` csharp
Span<int> numbers = stackalloc[] { 1, 2, 3, 4, 5, 6 };

// In earlier version:
// Span<int> searchNumbers = stackalloc[] { 2, 4, 6, 8 };
// var idx = numbers.IndexOfAny(searchNumbers);
// Now:
var idx = numbers.IndexOfAny(stackalloc[] { 2, 4, 6, 8 });
```

### 擴充內插逐字字串 (Enhancement of interpolated verbatim strings)
可支援 `$@"..."` 和 `@$"..."`，順序不會造成編譯錯誤。

### 參考
[What's new in C# 8.0 (原文件已經被刪除，這是 Archive 網站的存檔)](https://web.archive.org/web/20211203061114/https://docs.microsoft.com/en-gb/dotnet/csharp/whats-new/csharp-8)