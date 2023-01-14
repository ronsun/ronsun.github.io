---
title: 簡化 if 條件式
date: 2018-08-19 00:03:30
categories:
- C#
- Clean Code
tags:
---

`if` 條件式很常用, 但在有時因為使用不當造成閱讀或維護上的困難, 尤其是巢狀的 if-else 條件式對維護上造成很大的負擔, 這其實是有一些小技巧可以簡化 `if` 條件式的, 當然, 這些技巧不是萬靈丹, 使用上還是有些細節要注意的.

<!--more-->

### 一般 if 條件式
一般的 `if` 條件式其實沒什麼好簡化的, 但如果之後的**處理邏輯很簡單**或**滿足特定情境**的話, 還是有簡化的空間.

#### 三元運算子 `?:`
先附上[官方文件](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/conditional-operator).  

三元運算子是基本規格也沒啥好說的, 但如果用不好反而會降低可讀性.  

``` csharp
// string s;
// if(isTure)
// {
//     s = "Empty"
// }
// else
// {
//     s = foo;
// }
string s = isTrue ? "Empty" : foo;

// string s2;
// if(isTure)
// {
//     s2 = GetTheString();
// }
// else
// {
//     s2 = GetAnotherString();
// }
string s2 = isTure ?
            GetTheString() :
            GetAnotherString();

```

> 建議:  
> 1. 一般只有一行文的情境下才適用.
> 2. 用了他之後會比原本的 if-else 易讀就用.  
> 3. 上例中第二種分三行的做法要視情境斟酌一下, 不一定適合. 

> 避免:  
> 1. 使用兩層(或超過)的三元運算子, 因為可讀性很差.

#### Null 條件運算子 `?.`, `?[]` 與 `??`
先附上[`?.`, `?[]` 的官方文件](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-conditional-operators) 以及 [`??` 的官方文件](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator), 其中 `?.` 和 `?[]` 是 C# 6.0 之後的規格. 

主要用途是減少空值判斷的程式碼, 讓維護時更能專注於商業邏輯上.  
``` csharp
// string s;
// if(value == null)
// {
//     s = "Empty";
// }
// else
//     s = value;
// {
// }
string s = value ?? "Empty";

// string s2;
// if(obj == null)
// {
//     s2 = null;
// }
// else
// {
//     s2 = obj.Property;
// }
string s2 = obj?.Property;

// string s3;
// if(obj == null)
// {
//     s3 = "Empty";
// }
// else
// {
//     s3 =  obj.Property;
// }
string s3 = obj?.Property ?? "Empty";

// string s4;
// if(objectWithIndexer == null)
// {
//     s4 = null;
// }
// else
// {
//     s4 = objectWithIndexer["key"];
// }
string s4 = objectWithIndexer?["key"];
```

> 建議:  
> 1. 很好用, 可以盡量用.

### 巢狀 if 條件式
相信維護過有歷史包袱的系統的工程師多少都見過巢狀的 `if` 條件式, 甚至是巢狀的 `if`, `else if`, `else` 堆疊起來的程式碼片段, 這樣的程式碼常常看到後面忘了前面, 要花很多時間甚至畫出流程圖才能理解複雜的邏輯. 

#### 合併判斷條件
下面的程式碼是最單純的情境, 這類型的巢狀 `if` 條件式可以很輕易的合併.

``` csharp
// if(a)
// {
//     if(b)
//     {
//         if(c)
//         {
//             //Do something...
//         }
//     }
// }
bool isTrue = (a && b && c);
if(isTrue)
{
    //Do something...
}
```

> 建議:  
> 1. 實際情況當然不會這麼單純, 如果參雜 `else if` 或 `else` 的時候就很容易出錯, 先寫好單元測試再來合併條件式會比較好. 

#### 提早返回(return)
提早返回是一個我很喜歡用的作法, 但用不好的話可讀性還是不會太好.  

``` csharp
// if(a)
// {
//     // Do somthing...
// }
// else if(b)
// {
//     // Do somthing...
// }
// else
// {
//     // Do somthing
// }
if(a)
{
    // Do somthing...
    return;
}

if(b)
{
    // Do somthing...
    return;
}

// Do somthing...
return;
```

> 建議:  
> 1. 提早返回的行為盡可能集中在方法的最前面, 或只集中在同一個區塊, 這樣的思路就單純是在某些情境下提早返回, 否則往繼續執行.  
> 2. 提早返回前要確認條件式結束後沒有其他邏輯.

> 避免:  
> 1. 在一個方法中多次且分散的提早返回, 這樣維護時需要一直注意哪些地方有提早返回, 反而分散注意力了.  

#### 依職責拆分出不同方法
如果前面的方法都做了, 也避開了誤用的情境, 但還是存在巢狀的 `if` 條件式時, 就要考慮這個方法是否違反了單一職責原則(Single Responsibility Principle, SRP), 並考慮將某些情境重構成獨立的方法.  

### 結論
其實這些做法也不是什麼神奇的技巧, 但沒人提醒或是沒在專案中見過的話就很難意識到, 我剛入行時也是常常弄出巢狀的 `if` 條件式, 覺得對後面接手的人有點抱歉XD

### 參考
[官方文件](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/)
