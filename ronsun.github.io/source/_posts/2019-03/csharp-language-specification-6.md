---
title: C# 語言特性更新 - C# 6
date: 2019-03-22 23:26:12
categories:
- C#
- Language Spec
tags:
---

C# 的版本最近幾年更新的越來越頻繁, 雖然官方有提供[版本歷程](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-version-history)以及各版本的新特性, 但是還是想整理起來, 在官方提供的語言特性說明外多一點思考並紀錄自己使用上的想法.  

本篇是 C# 6.0

<!--more-->

### C# 6
#### 唯讀屬性 (Read-only auto-properties)
[Read-only auto-properties](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#read-only-auto-properties)  

``` csharp
public string Name { get; }
```

雖然舊版可以用 `public string Name { get; private set; }` 最大限度的限縮寫入權, 但仍然不是唯讀(只允許在建構子中改變屬性值), 硬要拆成私有唯讀欄位 (`private readonly fieldName`) 和 getter 方法的組合又失去了自動實作屬性的優點, 難免可惜.  

這個特性讓我們能更精細的控制屬性的存取範圍, 用在需要嚴謹的控制屬性的存取範圍的時候很好用.  

#### 屬性初始化器 (Auto-property initializers)
[Auto-property initializers](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#auto-property-initializers)  

``` csharp
public string Name { get; set; } = "Ron";
```

極相似於舊版在建構子中初始化屬性的做法, 舊版做法在建構子中初始化屬性的做法在屬性很多時, 排版上會影響可讀性.  
這個特性能讓屬性初始化更為簡潔, 更容易避免舊的做法的潛在問題.  

> 會說"極相似"從建構子初始化屬性代表還是有一點差異的  
> + 舊的做法: 從建構子初始化屬性其實是預設呼叫 setter 方法, 如果沒有 setter 方法則直接操作自動產生的欄位 (但舊版不支援唯讀屬性)
> + 新特性: 直接操作自動產生的欄位, 不管有沒有 setter 方法
> 
> 以上結論是透過 Msiler 套件產生 IL 碼後觀察的結果

#### 具有運算式主體的函式成員 (Expression-bodied function members)
[Expression-bodied function members](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#expression-bodied-function-members)  

讓只有單一運算式的方法或唯讀屬性可以用較短的程式碼完成.  

這個特性我平常其實不太用, 因為下面這三行的長相太相似了, 在閱讀上比較容易造成混淆, 且如果不小心用錯雖然不會造成編譯錯誤, 但含意卻會完全不同, 而增加的方便性給我的感受不是很強烈.  
``` csharp
public string FirstName => "Ron";
private string _lastName = "Sun";
public string GetFullName() => $"{FirstName} {_lastName}";
```

不小心手滑用錯的範例如下, 看起來很詭異, 但依然能正常運作:
``` csharp
// "=>" 變成 "=" 從唯讀屬性變成欄位
// FirstName 現在看起來像公開欄位, 是私有欄位寫錯變公開了?  
// 還是應該是唯讀屬性, 只是不小心少了一個 ">"?
public string FirstName = "Ron";

// "=" 變成 "=>" 從欄位變成唯讀屬性
private string _lastName => "Sun";

// 少了 "()" 從方法變成唯讀屬性
public string GetFullName => $"{FirstName} {_lastName}";
```

其實如果專案有良好的程式碼規範, 從命名基本上就可以分辨出這個成員應該要是欄位, 屬性還是方法, 加上工程師如果夠謹慎, 上面的失誤應該不會發生, 但假設專案沒有良好的規範又加上手滑打錯的狀況, 後面維護的人就可能會很混亂.

這不是說這個特性不好, 如果對團隊成員的嚴謹度與基礎有信心, 用這個特性也是很不錯的.  

#### using static
[using static](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#using-static)  

``` csharp
using static ConsoleApp1.Person;
class Program
{
    static void Main(string[] args)
    {
        var nestedPerson = new NestedPerson();
        var fullName = GetFullName();
    }
}

public class Person
{
    public static string GetFullName()
    {
        return "Ron Sun";
    }

    public class NestedPerson
    {
        public string Name { get; set; } = "Ron";
    }
}
```

以上面的例子來說, 舊的做法如果要初始化 `NestedModel` 要寫成 `new Model.NestedModel`, 呼叫靜態方法 `GetFullName()` 要寫成 `Model.GetFullName()`

這個特性我比較不喜歡, 主要是巢狀類別跟靜態方法這樣使用或呼叫的話, 外觀上跟一般類別或該類別內的方法呼叫一樣, 閱讀時如果沒有進去類別或方法中看, 難以分辨他究竟是不是該類別內部的方法.  

#### Null 條件運算子 (Null-conditional operators)
[Null-conditional operators](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#null-conditional-operators)  

``` csharp
string name = person?.Name;
string nameWithDefault = person?.Name ?? "John";
var company = person?.Team?.Company;
int? age = dictionary?["Ron"];
int ageWithDefault = dictionary?["Ron"] ?? -1;
```

用來省下空值判斷, 要取得物件中的屬性值的時候很好用, 也可以用在需要存取陣列或是索引子(indexer) 的情境.  

**這個特性神好用, 搭配 `??` 更神**, 可以縮短非常多空值判斷的程式碼, 尤其是像 `model.Nested.Property` 這種巢狀的存取更明顯, 就上面的範例來說, 舊的做法就明顯冗長, 如下面比較:

``` csharp
// string name = person?.Name;
string name = null;
if(person != null)
{
    name = person.Name;
}

// string nameWithDefault = person?.Name ?? "John";
string nameWithDefault = "John";
if(person != null)
{
    nameWithDefault = person.Name;
}

// Company company = person?.Team?.Company;
Company company = null;
if(person != null && person.Team != null)
{
    company = person.Team.Company;
}

// int? age = dictionary?["Ron"];
// int ageWithDefault = dictionary?["Ron"] ?? -1;
// ...依此類推
```

舊的做法的範例還是為了避免巢狀 if-else 而簡化過的, 如果是直覺式的 if-else 一層層科下去, 差異就更明顯了.  

但是在型別推論上會有一些陷阱，詳見 {% post_link pitfall-of-nullable-value-type %}。  

#### 字串插補 (String interpolation)
[String interpolation](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#string-interpolation)  

``` csharp
public string FullName => $"{FirstName} {LastName}";
```

**神好用, 真心不騙**, 這個範例看起來可能沒有感覺, 會覺得用 `string.Format()` 做也不錯啊, 但情境夠複雜的話無疑是海放最古典的字串相加, 也避免了 `string.Format()` 遇到大量參數的時候容易排錯順序的問題, 如果有用過組過 SOAP 用的 XML 內容的話, 就能明顯感受到它的威力了.  

另外, 搭配 `@` 使用, 效果更好.  

這個特性還有有很多進階用法, 官方資源如下:
+ [Use string interpolation to construct formatted strings](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/intro-to-csharp/interpolated-strings)  
+ [$ - string interpolation (C# Reference)](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated)
+ [String interpolation in C#](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation#how-to-specify-a-format-string-for-an-interpolated-expression)


#### 例外狀況篩選條件 (Exception filters)
[Exception filters](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#exception-filters)  

這個特性還沒機會用到, 不過目前看起來有兩個好處:  
+ 讓 Catch 區塊的情境更精細, 減少 Catch 區塊內的條件判斷.  
+ 攔截例外前多了一個機制可以判斷真的要攔截這個例外或是讓他繼續往外拋, 可以**避免掉有條件重拋例外所造成的 StackTrace 被改變的問題**, 重拋例外產生的問題之前寫過一篇 {% post_link correctly-re-throw-exception %}.  

#### nameof 運算式 (The nameof expression)
[The nameof expression](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#the-nameof-expression)  

``` csharp
// output: Method
Console.WriteLine(nameof(Class1.Method));

// output: Property
Console.WriteLine(nameof(Class1.Property));

// output: Property
Console.WriteLine(nameof(Class1.NestedClass.Property));
```

好東西, 這個特性比較偏可維護性的優化, 很在意可維護性的工程師會很愛用.  

需要在字串中放型別名稱, 方法名稱, 欄位名稱, 屬性名稱等等的情境時, 舊方法都是直接寫死字串, 這樣會導致重新命名的時候 visual studio 會將這些字串忽略掉, 老舊專案的 log 中就很常出現與實際不吻合的方法名稱.  

另外一個情境是反射, 如果專案中有用反射存取特定成員或呼叫方法, 在重新命名的時候往往會被忽略, 偏偏這種情境只能等到執行階段才會被發現, 導致重新命名風險提高, 但使用 `nameof` 之後, 可以避免這類問題, 重新命名時也可以更安心.  


#### Catch 和 Finally 區塊中的 Await
[Await in Catch and Finally blocks](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#await-in-catch-and-finally-blocks)  


#### Initialize associative collections using indexers
[Initialize associative collections using indexers](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#initialize-associative-collections-using-indexers)  

``` csharp
var dic = new Dictionary<string, string>()
{
    ["key1"] = "value1",
    ["key2"] = "value2",
};
```

舊的寫法是如下面這樣.  
``` csharp
var dic = new Dictionary<string, string>()
{
    { "key1", "value1" },
    { "key2", "value2" }
};
```

恩...乍看之下就是用換個初始化的語法, 但內部運作卻是不一樣的, 讓我們繼續看下去...  

透過工具看了一下編譯出來的 IL 碼發現兩者實際的運作並不同, 舊的做法呼叫的是 `Add(System.String,System.String)` 方法, 而 C# 6 的語法呼叫的是 `set_Item(System.String,System.String)`, 而這樣的差別會在某些情境下使得運作結果不同.   
``` il
// var dic = new Dictionary<string, string>()
// {
// { "key1", "value1" }
// };
IL_002B newobj    System.Void System.Collections.Generic.Dictionary`2<System.String,System.String>::.ctor()
IL_0030 dup
IL_0031 ldstr     "key1"
IL_0036 ldstr     "value1"
IL_003B callvirt  System.Void System.Collections.Generic.Dictionary`2<System.String,System.String>::Add(System.String,System.String)
IL_0040 nop
IL_0041 stloc.0
// var dic2 = new Dictionary<string, string>()
// {
// ["key1"] = "value1"
// };
IL_0042 newobj    System.Void System.Collections.Generic.Dictionary`2<System.String,System.String>::.ctor()
IL_0047 dup
IL_0048 ldstr     "key1"
IL_004D ldstr     "value1"
IL_0052 callvirt  System.Void System.Collections.Generic.Dictionary`2<System.String,System.String>::set_Item(System.String,System.String)
IL_0057 nop
IL_0058 stloc.1
```

根據官方說明, 新的語法是個 "Index Initializers", 搭配 [New Ways to Initialize Collections in C# 6](http://www.informit.com/articles/article.aspx?p=2424333) 這篇大神文章中的描述:

> There is one caution. The new initializer syntax calls the indexer method to add items to the collection. That same indexer method replaces items as well as adding items.  

可以發現新的寫法由於是直接操作 indexer, 所以**當索引 (key) 重複的時候會直接將值覆蓋, 不同於舊版做法的拋出例外**, 可以由下面兩段程式碼來比較兩者差異
``` csharp
// throw an exception because of duplicate key
var dic = new Dictionary<string, string>()
{
    { "k", "v1" },
    { "k", "v2" }
};
Console.WriteLine(dic["k"]);

var dic2 = new Dictionary<string, string>()
{
    ["k"] = "v1",
    ["k"] = "v2",
};
// the value should be "v2" because the second value of key "k" replace first one
Console.WriteLine(dic2["k"]);
```

#### Extension Add methods in collection initializers
[Extension Add methods in collection initializers](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#extension-add-methods-in-collection-initializers)  

這個又需要[大神文章](http://www.informit.com/articles/article.aspx?p=2424333)救援了.  

總之就是讓下方語法預設呼叫另外寫的擴充方法 `Add` 使得初始化能在語意上更為順暢.
``` csharp
var myCollection = new MyStringCollection()
{
    "Ron",
    "Sun"
}
```

#### Improved overload resolution
[Improved overload resolution](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#improved-overload-resolution)  


#### Deterministic compiler output
[Deterministic compiler output](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6#deterministic-compiler-output)

### 結論
本來以為對 C# 特性已經算熟了, 沒想到還是有不少平常沒用到就以為他不存或是一直有錯誤理解的特性, 看來要找個時間一個一個仔細看看了.  

### 參考
[New Ways to Initialize Collections in C# 6](http://www.informit.com/articles/article.aspx?p=2424333)  
[What's New in C# 6](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-6)

---
