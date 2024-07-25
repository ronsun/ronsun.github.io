---
title: 各種字串插補的花式用法
date: 2024-07-25 22:26:10
categories:
- C#
- Language Spec
tags:
---

字串插補是 C# 中常用的特性，但由於其多變的用法，導致在一般專案開發中沒能充分活用。這篇文章將統整各種使用情境並加以說明。  

因為字串插補編譯後的結果多樣，根據不同版本的 C# 可能有不同結果，這篇提到的編譯後結果只適用於該情境。  

<!--more-->
### 基本情境
``` csharp
string Demo(string s1, string s2)
{
    return $"s1: {s1}, s2: {s2}";
}
```

### Expression
因為 `:` 有特殊用途 (在後面的小節會提)，所以三元運算子這種將 `:` 用於不同於字串插補用途的情境要把整個表達式用 `()` 包起來。  

``` csharp
string Demo(string s1)
{
    return $"s1: {(s1?.Length > 0 ? s1 : "Empty")}";
}
```

### 換行與跳脫  
可依照原始字串格式輸出，但 C# 11 前後版本語法差異很大。跳脫部分和一般字串一樣，除了大括號是 
``` csharp
{{}}
```
之外。  

C# 11  
``` csharp
string Demo(string s1)
{
    return $$"""
{
    "s1": "{{s1}}"
}
""";
}
```

舊版：  
``` csharp
string Demo(string s1)
{
    return $@"{{
    ""s1"": ""{s1}""
}}";
}
```

``` csharp
string Demo(string s1)
{
    return $"{{\"s1\": \"{s1}\"}}";
}
```

### 多型別組合
#### 通常隱含呼叫 `ToString()` 轉換
``` csharp
string Demo(string s1, int i1)
{
    return $"s1: {s1}, i1: {i1}";
}
```

如果變數不是字串型別，"通常"會呼叫其 `ToString()` 方法來轉換成字串。

#### 例外： 指派給 `FormattableString` 型別的變數時
[參考 How to create a culture-specific result string with string interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation#how-to-create-a-culture-specific-result-string-with-string-interpolation)  

``` csharp
using System.Globalization;

string Demo(decimal d1, string culture)
{
    FormattableString formattable = $"The value is {d1:N}";
    string formatted = formattable.ToString(CultureInfo.GetCultureInfo(culture));
    return formatted;
}

Demo(1234.56m, "en-US").Dump(); // The value is 1,234.560
Demo(1234.56m, "fr-FR").Dump(); // The value is 1 234,560
Demo(1234.56m, "de-DE").Dump(); // The value is 1.234,560
```

編譯後的程式碼不像表面語法的那樣會馬上執行字串插補，而是改為呼叫 `FormattableStringFactory.Create ("The value is {0:N}", d1)`，等到呼叫 `formattable.ToString()` 時才執行並參考 `CultureInfo`，實際上是這樣運作的：  
``` csharp
private string Demo(decimal d1, string culture)
{
	FormattableString formattable = FormattableStringFactory.Create("The value is {0:N}", d1);
	return formattable.ToString(CultureInfo.GetCultureInfo(culture));
}
```

#### 例外： 指派給 `FormattableString` 型別的變數時 (.NET 6)
[參考 How to create a culture-specific result string with string interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation#how-to-create-a-culture-specific-result-string-with-string-interpolation)  

``` csharp
using System.Globalization;

string Demo(decimal d1, string culture)
{
    return string.Create(CultureInfo.GetCultureInfo(culture), $"The value is {d1:N}");
}

Demo(1234.56m, "en-US").Dump(); // The value is 1,234.560
Demo(1234.56m, "fr-FR").Dump(); // The value is 1 234,560
Demo(1234.56m, "de-DE").Dump(); // The value is 1.234,560
```

.NET 6 之後可以簡化成上面這樣，但是編譯後的結果和之前的版本不一樣，是改使用 `DefaultInterpolatedStringHandler` 如下：  
``` csharp
private string Demo(decimal d1, string culture)
{
	IFormatProvider cultureInfo = CultureInfo.GetCultureInfo(culture);
	DefaultInterpolatedStringHandler val = default(DefaultInterpolatedStringHandler);
	((DefaultInterpolatedStringHandler)(ref val))..ctor(13, 1, cultureInfo);
	((DefaultInterpolatedStringHandler)(ref val)).AppendLiteral("The value is ");
	((DefaultInterpolatedStringHandler)(ref val)).AppendFormatted<decimal>(d1, "N");
	return string.Create(cultureInfo, ref val);
}
```

#### 例外： IFormattable
[參考 Compilation of interpolated strings](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated#compilation-of-interpolated-strings)  

``` csharp
public class CustomType : IFormattable
{
    public string ToString(string format, IFormatProvider formatProvider)
    {
        return "CustomFormattedString";
    }

    public override string ToString()
    {
        return "DefaultString";
    }
}

string Demo()
{
    var custom = new CustomType();
    return $"Formatted: {custom}";
}

Demo().Dump(); // Formatted: CustomFormattedString
```

實作 `IFormattable` 的型別在插補字串中，因為呼叫了 `string.Format()` 或是使用 `DefaultInterpolatedStringHandler` (因 C# 與 .NET 版本不同而異) 最後是使用 `IFormattable` 提供的 `ToString(...)` 方法而不是 `object.ToString()`。  

### 空值防呆
``` csharp
string Demo(string s1, int? i1)
{
    return $"s1: {s1}, i1: {i1}";
}

Demo(null, null).Dump(); // s1: , i1: 
```

空值不會拋例外，不管編譯時視語言版本編譯成 `string.Format` 或 `DefaultInterpolatedStringHandler` 都支援空值。  

### 對齊與格式化
[細節參考 Structure of an interpolated string 中的各種連結](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated#structure-of-an-interpolated-string)  

``` csharp
$"Align:{"Ron",5}{100,4}".Dump(); // Align:  Ron 100
$"{DateTime.Now:yyyy-MM-dd}".Dump(); // 2024-07-26
$"{123.4:N2}".Dump(); // 123.40
$"{Guid.NewGuid():N}".Dump(); // a6f3649725924db881223b8ab4ef7c60
$"{TimeSpan.Parse("1.02:03:04.567"):g}".Dump(); // 1:2:03:04.567
```

### 結論
其實字串插補經過分類後並不會太複雜，只是因為各種情境下的分支細節太多，又有一些特例導致

### 參考
[String interpolation in C#](https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation)  

[String interpolation using `$`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated)  

[Improved Interpolated Strings](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/improved-interpolated-strings)  