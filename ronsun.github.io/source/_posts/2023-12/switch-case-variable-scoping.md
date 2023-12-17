---
title: 在 Switch-Case 中使用重名區域變數與延伸探討
date: 2023-12-17 23:50:27
categories:
- C#
- Language Spec
tags:
---

問題起始於一個情境如下：
``` csharp
switch (cryptoType)
{
    case CryptoType.AesEncrypt:
        var key = "";
        var iv = "";
        break;
    case CryptoType.AesDecrypt:
        var key = "";
        var iv = "";
        break;
	// Others
    default:
        break;
}
```

這時候因為 case 之間的區域變數名字重複了所以編譯不會通過。

> 現實情境更複雜，`CryptoType` 可能有幾十個。  
> 
> 這邊要先聲明，理論上這種大量的 switch-case 有很多設計可以管理它，但實務上是一個老舊又缺乏架構的專案，甚至亂到重構後的檔案放哪都只會更亂的程度，而如果只是抽出方法，那一些簡單的情境就會因此被抽出大量小型方法，且都放在同一個類別中也不會比較好。  
>
> 總之，這個例子只是拿來說明區域變數與其作用域和生命週期，不表示例子中的設計是恰當的。

<!--more-->

### 使用區塊限定作用域
在 cases 中建立利用區塊 `{}` 建立獨立的作用域：  

``` csharp
switch (cryptoType)
{
    case CryptoType.AesEncrypt:
        {
            var key = "";
            var iv = "";
        }
        break;
    case CryptoType.AesDecrypt:
        {
            var key = "";
            var iv = "";
        }
        break;
	// Others
    default:
        break;
}
```

### 區塊 (Block) 與區域變數生命週期  
在 C# 中，區塊 `{}` 不僅是 `if`、`for` 等控制流語句的一部分，也是作用域的界定者。在一個區塊內定義的區域變數，其作用域限定在該區塊內。這意味著同一變數名可以在不同區塊中獨立使用，不會相互影響。

另外如果有必要的時候，使用區塊來限定變數的作用域，不僅有助於解決命名衝突，還可以讓區域變數因為結束區塊而釋放。 但是這樣做會犧牲很大的可讀性與可維護性去得到微乎其微的效益，只能說 "理論上可以這樣做"。

另外，基於好奇稍微去看了 C# 的語言規格書怎麼描述這種基礎到不知道怎麼描述的語法，才發現其實這些看似基礎的東西，細節還真的不少，例如： [區域變數](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/variables#929-local-variables)、[區塊](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/statements#133-blocks)。

### 結論
其實單獨用區塊控制區域變數的生命週期就是一句話的事，實在很猶豫要不要寫成一篇。 但實務上可能還是會遇到要維護肥大的 switch-case 又無法改設計的情境，導致變數名稱常常重複，如果不用區塊分隔開就會出現為了避免區域變數重名而產生冗長難閱讀的變數，也會讓變數命名時還要去考慮現存的 case 中會不會出現相同的名字造成細微卻常態出現的干擾。 想想還是寫一篇紀錄一下好了。

### 參考
[C# standard specification](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/specifications)  