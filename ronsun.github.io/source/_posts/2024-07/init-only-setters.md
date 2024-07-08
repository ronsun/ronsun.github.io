---
title: 深入 Init Only Setters
date: 2024-07-11 00:13:50
categories:
- C#
- Language Spec
tags:
---

C# 提供許多編譯期的語法糖，但有些特性並不能完全歸類為語法糖，例如 Init Only Setters。本篇將快速介紹這個特性，並深入探討其運作原理。

<!--more-->

### 特性說明
這個特性精確來說是屬性**只能在建構子中賦值**，一旦離開建構子就不能再賦值了，而不是從字面解讀成 "只能賦值一次"。  

使用初始設定式可以賦值 (初始設定式編譯後是在建構子中賦值)：
``` csharp
public class Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
}

var ron = new Person
{
   FirstName = "Ron",
   LastName = "Sun",
};
```

建構子中賦值，甚至可以修改 (所以嚴格來說不是真的 Init-Only)：  
``` csharp
public class Person
{
    public Person(string firstName, string lastName, int age)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
        if(age > 17)
        {
            FirstName = firstName + lastName;
            LastName = null;
        }
    }
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
}
```

> 看起來也是為了 Immutable 所做，用在不想被偷改屬性值的情境還滿好用的。例如隨意修改傳入方法的參數中的屬性，再把該參數傳遞到其他方法中，最後難以追蹤參數值的變化。

### 背後機制
在 {% post_link auto-implemented-properties %} 這篇有介紹到屬性其實是一個編譯時期的語法糖，而 `init` 做為 setter 的修飾詞，那他背後怎麼運作想必也值得研究。  

要深入探討他背後的機制，第一步先從反組譯範例程式碼開始
``` csharp
public class Demo
{
    public int InitOnlyNumber { get; init; }
    public int GeneralNumber { get; set; }
}
```

反組譯到 C# 9 以前還未支援的版本，看看會變什麼樣子
``` csharp
// Untitled, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null
// Program.Demo
using System.Diagnostics;

public class Demo
{
	[field: DebuggerBrowsable(/*Could not decode attribute arguments.*/)]
	public int InitOnlyNumber { get; set/*init*/; }

	[field: DebuggerBrowsable(/*Could not decode attribute arguments.*/)]
	public int GeneralNumber { get; set; }
}
```

這個結果有點出乎意料，怎麼竟然只剩一個註解？既然從舊版 C# 看不出端倪，那乾脆轉成 IL 看看能不能看出差異好了，由於轉成 IL 之後非常冗長，這邊為了展示只擷取 setter 的部分來比較  

**set_GeneralNumber 部分：**
``` csharp
.method public hidebysig specialname 
	instance void set_GeneralNumber (
		int32 'value'
	) cil managed 
{
	.custom instance void [System.Runtime]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() = (
		01 00 00 00
	)
	// Method begins at RVA 0x20de
	// Header size: 1
	// Code size: 8 (0x8)
	.maxstack 8

	// <GeneralNumber>k__BackingField = value;
	IL_0000: ldarg.0
	IL_0001: ldarg.1
	IL_0002: stfld int32 Program/Demo::'<GeneralNumber>k__BackingField'
	// }
	IL_0007: ret
} // end of method Demo::set_GeneralNumber
```

**set_InitOnlyNumber 部分：**
``` csharp
.method public hidebysig specialname 
	instance void modreq([System.Runtime]System.Runtime.CompilerServices.IsExternalInit) set_InitOnlyNumber (
		int32 'value'
	) cil managed 
{
	.custom instance void [System.Runtime]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() = (
		01 00 00 00
	)
	// Method begins at RVA 0x20cd
	// Header size: 1
	// Code size: 8 (0x8)
	.maxstack 8

	// <InitOnlyNumber>k__BackingField = value;
	IL_0000: ldarg.0
	IL_0001: ldarg.1
	IL_0002: stfld int32 Program/Demo::'<InitOnlyNumber>k__BackingField'
	// }
	IL_0007: ret
} // end of method Demo::set_InitOnlyNumber
```

從上面的比對可以看到，Init Only Setters 轉換成 IL 只多了一個 `modreq([System.Runtime]System.Runtime.CompilerServices.IsExternalInit)`，這很可能就是其中的關鍵。  

接下來從 [Init Only Setters 設計書](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/init#modreqs-vs-attributes) 中可以了解在編譯時期插入 `modreq` 的原因，可惜這篇設計書沒提到 `modreq([System.Runtime]System.Runtime.CompilerServices.IsExternalInit)` 在執行階段的運作機制，但這篇 [StackOverflow](https://stackoverflow.com/a/77111055) 的回應有引用到的 [CLI 規格書](https://www.ecma-international.org/wp-content/uploads/ECMA-335_6th_edition_june_2012.pdf) 中可能有提到。  

**到這裡已經可以確定就是藉由 `modreq([System.Runtime]System.Runtime.CompilerServices.IsExternalInit)` 來插入 `IsExternalInit`，使得執行時能達到僅限初始化時賦值的效果。**  

另外設計書中所提到的 "Compilers unaware of `init` will ignore the `set` accessor..." 一直讓我覺得很困惑，編譯器不認識 `init` 應該會編譯失敗，怎麼會是忽略 `set`？ 這部分在 [C# 9 Records and Init Only Settings Without .NET 5](https://btburnett.com/csharp/2020/12/11/csharp-9-records-and-init-only-setters-without-dotnet5.html) 這篇文章中 Consuming Records and Init Only PropertiesPermalink 這段或許可以說明，就是套件支援 C# 9 但依賴套件的專案不支援時，對用戶端而言會直接把屬性視為唯讀屬性。  
> 這部分我有想試著重現，結果是呼叫 setter 時會因為 C# 版本不一致而編譯失敗 (CS8400	Feature 'init-only setters' is not available in C# 8.0. Please use language version 9.0 or greater.)，而 getter 可以正常使用，和上面這篇文章所說的 "把屬性視為唯讀屬性" 不完全一致 (也可能他語意上就是想表達這個現象)。

### 漏洞
從上一段我們知道了 Init Only Setter 的機制，也從設計書看到他的缺點。   

#### 反射
無法阻止透過反射修改屬性值。  
``` csharp
public class Demo
{
    public int InitOnlyNumber { get; init; }
}

var x = new Demo() { InitOnlyNumber = 100 };

// Reflection to change the value of init only property.
typeof(Demo).GetProperty("InitOnlyNumber").SetValue(x, 1);

// Changed to 1.
x.InitOnlyNumber.Dump();
```

#### `dynamic` (無法重現)  
可能在我的環境中用的是已經修好這個問題的編譯器或相關工具。  

#### 不認識 modreqs 的編譯器
在套件開發上要注意，如果套件中使用 `init` 後編譯成 dll 被其它專案引用，而這個專案使用不支援 C# 9 的編譯器，就會因為無法辨識 Init Only Setter 而使得套件中的相關功能無法使用。  

### 衍伸知識與關鍵字
從參考資料中我們會另外看到一些衍伸的關鍵字，如果不懂這些關鍵字會很難理解設計書中的內容。  

**binary signature**： 根據 ChatGPT 說明，在 .NET 中每個方法或屬性都有一個二進位簽名（binary signature），用來唯一標識這個方法或屬性。這個簽名包括方法或屬性的名稱、參數類型、返回類型以及其他修飾符。  

**binary compatbility**： 編譯後的程式碼是否能與已經編譯好的其他程式碼相互合作。  
如果一個方法或屬性的二進位簽名發生改變，現有的二進位相容性就會被破壞，因為其他程式碼可能依賴於這個特定的簽名。也就是說當一個屬性將 `set` 改成 `init` (反之亦然)時，即便修改前後所產生的 IL 中都有 setter 二進位簽名也會因為 `modreq` 而改變，這時候其它編譯好的程式碼對這個屬性的參考，就會因為二進位簽名改變而無法正確定位。

**CLI、IL**： 這個網路查查很多，但是比較不容易理解的是這些詞之間的關係，簡單說 IL 是 CLI 標準的一部份，[Ecma335、CLR、CLI、CTS、 IL、.net 以及他们之间的关系](https://www.cnblogs.com/cdaniu/p/15147320.html)有一些說明。

### 結論
其實主要就是編譯時期插入一些 IL 程式碼使得執行階段能辨識並如預期般運作。比較要注意的是套件開發者現在除了考慮框架版本的支援度外，也要考慮語言版本支援度的問題了。  

### 參考
[关于C#9中仅初始化的属性设置器](https://miroox.github.io/blog/2021/05/Initonly-Setter-CSharp/)  

[Init Only Setters](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/init)：尤其是 **Modreqs vs. attributes** 和 **Metadata encoding** 兩段。

[Why is changing a property from "init" to "set" a binary breaking change? 的回應](https://stackoverflow.com/a/77111055)  

[CLI 規格書](https://www.ecma-international.org/wp-content/uploads/ECMA-335_6th_edition_june_2012.pdf)  

[Ecma335、CLR、CLI、CTS、 IL、.net 以及他们之间的关系](https://www.cnblogs.com/cdaniu/p/15147320.html)  

[C# 9 Records and Init Only Settings Without .NET 5](https://btburnett.com/csharp/2020/12/11/csharp-9-records-and-init-only-setters-without-dotnet5.html)