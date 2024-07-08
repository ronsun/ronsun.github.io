---
title: C# 語言特性更新 - C# 9
date: 2024-06-24 23:52:53
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->
### `record` (Record types)
用更簡潔的語法定義類別，屬於參考型別，可定義可變 (Mutable) 屬性與不可變 (Immutable) 屬性。雖然如此，但 Record 主要目的還是用於不可變屬性的。  

``` csharp
// The FirstName and the LastName are immutable properties,
// whereas the Age is mutable property.
public record Person(string FirstName, string LastName)
{
    public int Age { get; set; }
}

// Create with immutable properties.
var ron = new Person("Ron", "Sun");
// Modify the mutable property.
ron.Age = 100;
```

> Record 支援可變屬性比較像是為了提高支援度，畢竟很多時候一個幾乎都是不可變屬性的 DTO 就是可能包含幾個可變屬性。

還有一些特性：
+ 支援位置語法 (Positional Syntax)，例如：`Person ron = new("Ron", "Sun");`。  
+ 實值相等 (Value equality)，當型別相同且所有欄位與屬性都相同時，兩個變數可以用 `==` 來判定相等，這和一般參考型別不同。
+ 非破壞性變化 (Nondestructive mutation)，使用 `with` 關鍵字來複製物件並賦值要變化的內容，例如：`Person newRon = ron with { FirstName = "Ron 2.0" };`。
+ 內建的格式化 (Built-in formatting for display)，透過 `ToString()` 方法能得到一個預設的格式化字串，例如：`Person { FirstName = Ron, LastName = Sun, Age = 0 }`。
+ 繼承的限制 (Inheritance)，Record 之間可以繼承，但 Record 和 class 互相不能繼承。  

> 整體看起來除了簡化語法外，另外也想用參考型別來模擬實值型別的特性，來得到兩者的優點。並且提供一些預設特性來達到方便使用的效果，但因為這些設計造成他和一般類別有種非常相似卻又有很多不同的情況，之後需要就 Record 這個主題另外深入探討。

### 僅供初始化的 Setter (Init only setters)
詳見 {% post_link init-only-setters %}。 

### 最上層陳述式 (Top-level statements)
省略 `Prgram` 類別和 `Main` 方法的大部分程式碼。

> 看起來有點雞肋，實際專案沒甚麼效益，但用 RoslynPad 等工具做實驗時不用再把整個 `Main` 方法刻出來有比較方便。


### 更多的模式比對 (Pattern matching enhancements)
括號、`and`、`or`、`not`、`>`、`<`、`=` 等，如下綜合範例。

``` csharp
public static bool IsLetterOrSeparator(this char c) =>
    c is (>= 'a' and <= 'z') or (>= 'A' and <= 'Z') or '.' or ',';
```

> 這符號實在太多了，果然還是需要另外開一篇文章詳細介紹。

### Performance and interop

### Fit and finish features
1. 基於 `new` 關鍵字的推論型別。
    ``` csharp
    Person p = new();
    InitPerson(new(), "Ron", "Sun");
    // void InitPerson(Person p, string firstName, string lastName)
    ```
2. `static` 修飾詞可加在在 lambda 運算式或匿名方法上。
3. 其他看不太懂的之後再研究。

### 程式碼產生器的支援

### 結論
C# 9 有一些比較新特性衍生出很多可以研究的細節，這篇先概略性的紀錄一下，之後根據特性再另外深究發文。  

> 已經跟不太上了。

### 參考
[What's new in C# 9.0 (原文件已經被刪除，這是 Archive 網站的存檔)](https://web.archive.org/web/20211203041103/https://docs.microsoft.com/zh-tw/dotnet/csharp/whats-new/csharp-9)  

ChatGPT