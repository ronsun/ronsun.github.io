---
title: 可空的實質型別與其陷阱
date: 2024-07-16 23:50:00
categories:
- C#
- Language Spec
tags:
---

`Nullable<T>` 是一個很方便的結構，但看似簡單的特性背後卻隱藏一些小陷阱，這些陷阱平時難以發現，但一旦遇到卻很容易讓人感到困惑，這篇文章將解析我遇過的相關情境。

<!--more-->

### 裝箱與 `GetType()` 時型別不如預期
``` csharp
decimal? d = null;
decimal? d2 = 1m;

// Type dType = d.GetType(); // Exception
Type d2Type = d2.GetType();
d2Type.FullName.Dump(); // System.Decimal
typeof(decimal?).FullName.Dump(); // System.Nullable`1[[System.Decimal, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]
```

從上面範例可以看到，兩個變數都是 `Nullable<Decimal>` 型別，但神奇的是 `dType` 那一行會拋錯而 `d2.GetType()` 那一行卻會回傳 `Decimal` 這個型別；而另一方面直接使用 `typeof(decimal?)` 卻如預期的回傳 `Nullable<Decimal>`。  

根據[官方文件 How to identify a nullable value type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#how-to-identify-a-nullable-value-type) 一節，這件事得從裝箱 (Boxing) 說起。首先，`d2.GetType()` 這一行會先轉型成 `Object` 並引發裝箱，而有值的 `Nullable<Decimal>` 裝箱的行為實際上是將內部的基礎型別 (`Decimal`) 的值裝箱，以至於 `d2Type` 的內容其實是 `Decimal`。由於本質上是因為裝箱引發的結果，這件事不只會發生在呼叫 `GetType()` 時，也會在把 `Nullable<T>` 轉型成 `object` 而引發裝箱時產生一樣的現象。  

另外根據 CLR via C# 這本書第19章 Nullable Value Types 中 The CLR Has Special Support for Nullable Value Types 一節的說法，CLR 有針對 `Nullable<T>` 做特殊處理，因此我們無法透過模仿 `Nullable<T>` 的設計而寫出一個客製化的型別來達到一樣的特性。  

更甚至在型別判斷上也會出現類似的誤判：
``` csharp
decimal? d2 = 1m;
(d2 is decimal?).Dump(); // True
(d2 is decimal).Dump(); // True
```

因此如果要偵測一個物件是不是 `Nullable<T>`，就只能靠下面的輔助方法來達成：
``` csharp
static bool IsNullableType<T>(T instance)
{
    Type type = typeof(T);
    return Nullable.GetUnderlyingType(type) != null;
}

decimal? d2 = 1m;
IsNullableType(d2).Dump(); // True
IsNullableType(1m).Dump(); // False
```

### 預期外的推論型別結果
搭配三元運算子 (Ternary Operator) 和 Null 聯合運算子 (null-coalescing operators) 使用時， `var` 的推論型別可能和預期的不一樣。  

以下面的範例為例：
``` csharp
decimal? d = null;
var x = d.HasValue ? d : 1m; // Nullable<Decimal>
var x2 = d ?? 1m; // Decimal
```

首先三元運算子的部分 `var x = d.HasValue ? d : 1m;` 其實不太意外畢竟 `d` 是 `Nullable<Decimal>` 型別而 `1m` 的部分是 `Decimal`，將 `x` 推論成支援範圍比較廣的 `Nullable<Decimal>` 很合理。  

但 Null 聯合運算子部分就令人意外了。 下面提供兩種不同的工具反組譯出來的結果，雖然細節不大一樣，但關鍵點都一樣是因為 `Nullable<T>.GetValueOrDefault()` 回傳的型別是 `T`，以至於推論型別只會是 `Decimal`。  
``` csharp
// From ILSpy
Nullable<decimal> d = null;
Nullable<decimal> num = d;
decimal x2 = (num.HasValue ? num.GetValueOrDefault() : 1m);

// From LINQPad 8
Nullable<decimal> d = null;
decimal x2 = d.GetValueOrDefault (1m);
```

這邊另外補充一下 ChatGPT 的解釋與參考資料：
> 1. C# 編譯器在推斷三元運算子結果型別時，會考慮兩個操作數的最小共同超類型 (common type) 。如果操作數之一是 `Nullable<T>` 而另一個是 `T`，則結果會被推斷為 `Nullable<T>`，因為 `Nullable<T>` 可以包含所有 `T` 的值和一個額外的 `null`。參考 [?: operator - the ternary conditional operator](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/conditional-operator)  
> 2. `??` 運算子用於在其左操作數為 `null` 時返回其右操作數。由於 `Nullable<T>` 可以被看作兩種狀態(有值和無值)，在運行 `??` 時，如果左操作數是 `null`，則直接返回右操作數的值類型。參考 [?? and ??= operators - the null-coalescing operators](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)

### 結論
C# 很多由編譯器處理的語法糖，如果不了解編譯後的結果，有時候會對一些細節產生誤判。

### 參考
ChatGPT  

CLR via C# 一書

[How to identify a nullable value type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#how-to-identify-a-nullable-value-type)  

[?: operator - the ternary conditional operator](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/conditional-operator)  

[?? and ??= operators - the null-coalescing operators](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)  