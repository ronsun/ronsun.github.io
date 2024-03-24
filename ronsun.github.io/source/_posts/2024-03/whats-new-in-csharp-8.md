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

### 唯獨成員 (Readonly members)
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


> 建議:  
> 1. ........

> 避免:  
> 1. ........

### Title2
#### Title2-1

### 結論

### 參考
[What's new in C# 8.0 (原文件已經被刪除，這是 Archive 網站的存檔)](https://web.archive.org/web/20211203061114/https://docs.microsoft.com/en-gb/dotnet/csharp/whats-new/csharp-8)