---
title: 消除switch-case
date: 2018-04-12 23:51:21
categories:
- C#
- Clean Code
tags:
---

Switch-case 是一個經常被使用到的語句, 但當條件/情境過多的時候, 他會變得很肥大, 某種程度上造成維護上的困擾, 網路上也有不少人提出針對過大的 switch-case 語句的重構技巧, 例如: 策略模式.  
但是策略模式會長出很多新的類別, 如果每個 case 的實作內容都很小, 這樣做似乎是有點複雜了, 所以這邊提供其他的替代方案(但會有一些限制).  

<!--more-->

### switch-case的問題

先看一段程式碼
``` csharp
public string Convert(StringConverter converter, string input)
{
    switch (converter)
    {
        case StringConverter.Rule1:
            return DoByRule1(input);
            break;
        case StringConverter.Rule2:
            return DoByRule2(input);
            break;
        case StringConverter.Rule3:
            return DoByRule3(input);
        //以下省略100種
        default:
            return string.Empty;
            break;
    }
}

public void Main()
{
    // ...
    string result = Convert(StringConverter.Rule1, "this is input");
    // ...
}
```

上面的程式碼是經過最初步整理的樣子, 至少每個case都提取出一個方法, 但是如果情境持續增加, 這個switch-case語句仍然會繼續成長下去, 似乎還能做一些改善.

### 用 Dictionary 搭配 Func 或 Action 解決
從原本的 switch-case 可以看出功能其實很單純, 固定傳入一個字串, 經過某種演算過程, 回傳另外一個字串, 只不過根據列舉(enum)的不同, 要做的事也不同, 這種情境就很適合用 `Dictionary<TKey, Func<>>` 來處理.

``` csharp
private Dictionary<StringConverter, Func<string, string>> allRules =
    new Dictionary<StringConverter, Func<string, string>>()
    {
        [StringConverter.Rule1] = (input) => { return DoByRule1(input); },
        [StringConverter.Rule2] = (input) => { return DoByRule2(input); },
        [StringConverter.Rule3] = (input) => { return DoByRule3(input); }
        //以下省略100種
    };

public void Main()
{
    // ...
    var rule = allRules[StringConverter.Rule1];
    string result = rule("this is input");
    // ...
}
```

經過整理後, 我們將列舉與行為的配對抽出來成為一個 `Dictionary<StringConverter, Func<string, string>>`, 可讀性提高了不少(尤其 case 很多的時候會更明顯).  
但是這種方式是有一些限制的, 所有情境必須有相同的參數與回傳型別, 才能把他們全部放到一個 `Dictionary` 中, 如果不幸的所些情境參數或回傳型別不同, 那就要考慮是否另外封裝共用的物件作為參數與回傳型別, 此舉是有副作用的(共用的參數與回傳型別中的成員是所有情境的聯集, 可能會很多或很亂), 因此也應該考慮這樣做的效益是否大於副作用.  

搭配 Action 概念跟上一個例子一樣, 不過是用無參數的 Action.  

### 結論
技巧跟手法有好處就有副作用, 所以不是所有 switch-case 的情境都適合這樣處理, 有些情境可能維持原本的 switch-case 會更好.  

> 類似的技巧還有用 `List<Func<>>` 搭配 `foreach`, 可用來達成依序執行一系列的行為的目的, 也是很不錯

### 參考
[Abolishing Switch-Case Statement and Pattern Matching in C# 7.0](https://www.codeproject.com/Articles/1192465/Abolishing-Switch-Case-Statement-and-Pattern-Match)
