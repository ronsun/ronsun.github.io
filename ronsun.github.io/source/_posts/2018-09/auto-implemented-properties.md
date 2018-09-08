---
title: C# 的自動實作屬性
date: 2018-09-09 00:08:45
categories:
- C#
- Language Spec
tags:
---

自動實作屬性(auto implemented properties)是 C# 非常基本的規格, 雖然他跟 欄位(fileds) 使用上非常相似, 但本質上是不一樣的東西.  
這篇主要是紀錄一些網路上的比較資訊以及自己好奇下做的一個小實驗, 沒什麼太特別的內容. 

<!--more-->

### 欄位, 屬性與自動實作屬性
先來看看官方對於
[欄位(fileds)](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/fields), [屬性(properties)](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/properties)與[自動實作屬性(auto implemented properties)](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/auto-implemented-properties)的介紹, 這部分的介紹與比較網路資料多到看不完, 我就懶得再多整理一次了. 

> 建議:  
> 1. private 時使用欄位, 除此之外使用自動實作屬性. 
> 2. 如果需要對欄位進行自動實作屬性的預設行為以外的操作時, 使用屬性. 

### 自動實作屬性編譯後變什麼?
根據官方與大量網路資料的說法, 自動實作屬性編譯後會產生兩個存(set)取(get)用的方法和一個欄位(backing field), 昨天剛好好奇想說能不能知道這兩個方法和欄位究竟長什麼樣子.   
於是基於好奇, 就利用反射的方式在監看式(Watch)中去挖裡面的內容, 果然找到一些有用的東西, 整理後如下:
``` csharp
public class MyClass
{
    public string MyProperty { get; set; }
}

static void Main(string[] args)
{
    var property = typeof(MyClass).GetProperty("MyProperty");
    var runtimeField = typeof(MyClass).GetRuntimeFields().First();

    // method name for get: get_MyProperty
    Console.WriteLine("method name for get: " + property.GetMethod.Name);

    //method name for set: set_MyProperty
    Console.WriteLine("method name for set: " + property.SetMethod.Name);

    // name for backing filed: <MyProperty>k__BackingField
    Console.WriteLine("name for backing filed: " + runtimeField.Name);

    Console.ReadLine();
}
```

既然編譯後會產生這樣的兩個方法, 那如果我在 class 裡面刻意加上這兩個方法的話會如何呢?  
``` csharp
public class MyClass
{
    public string MyProperty { get; set; }

    public string get_MyProperty()
    {
        return null;
    }

    public void set_MyProperty(string val)
    {
    }
}
```

果然, 編譯不會過, 出現這樣的錯誤  
{% asset_img 001.png %}  

> 本來還想試試看能不能弄出跟 backing field 衝突的欄位, 不過失敗了.

其實這個實驗對於實作上沒什麼明顯的幫助, 不過有時候稍微追根究底一下也是有趣.  

### 結論
其實欄位, 屬性與自動實作屬性的使用時機還滿明確的, 以往使用上也沒有特別難以抉擇的情境出現, 不過真的要回答 "為什麼這樣用?" 時, 還真回答不出來, 然後這邊順便放一下網路上對於這部分的文章  

+ [C# in Depth - Why Properties Matter](http://csharpindepth.com/Articles/Chapter8/PropertiesMatter.aspx)  
+ [使用 屬性(Property) 的好處](https://dotblogs.com.tw/yc421206/archive/2011/06/06/27233.aspx)  
+ [Difference between Auto - Implemented Properties and normal public member variables](https://social.msdn.microsoft.com/Forums/en-US/d479ffff-41da-40fe-9274-62a211ef3edd/difference-between-auto-implemented-properties-and-normal-public-member-variables?forum=csharplanguage)  
+ [Is it bad practice to use public fields?](https://softwareengineering.stackexchange.com/questions/161303/is-it-bad-practice-to-use-public-fields)
+ [Automated property with getter only, can be set, why?](https://stackoverflow.com/questions/34743533/automated-property-with-getter-only-can-be-set-why)
