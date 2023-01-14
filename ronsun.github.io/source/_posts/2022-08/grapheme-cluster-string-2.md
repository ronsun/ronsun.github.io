---
title: 依字位 (grapheme) 處理字串 (2)
date: 2022-08-28 23:20:48
categories:
- C#
- .NET
tags:
---

前陣子在 {% post_link grapheme-cluster-string %} 這篇文章中有以越南文為例提到 grapheme 的問題，後來覺得 `TextElementEnumerator` 提供的方法不夠多，所以基於他另外封裝了一個仿 `String` 的類別，折騰了幾天常用的都寫得差不多了才發現繞遠路做了不少原本 `string` 就有提供的功能。  

<!--more-->

### 主要問題
主要問題是因為 unicode 的幾種不同的標準化格式，導致字面上一樣的字其實是不同的內容，造成字串在比對的時候 (`==` 運算子) 會認為雙方是不同的字。  

關於標準化格式就放連結在這邊就好，要看再慢慢看:  
+ [UNICODE NORMALIZATION FORMS](https://unicode.org/reports/tr15/)  
+ [Unicode Normalization 文字標準化](https://blog.sean.taipei/2021/12/unicode)  

### 解決方案
#### `String`
其實 `String` 提供的不少方法就有包含到這個問題，通常透過方法中的 `StringComparison` 或 `CultureInfo` 參數來做。

#### `String.Normalize(...)`
這個方法可以將字串轉為特定的標準化格式，轉換後可以直接比對字串不需要考慮格式問題

> 避免用 `String.Normalize(...)` 做字串取代後輸出，轉化後的內容和輸入內容不同這樣做容易出意外，例如: 預期沒有要取代的內容卻被轉化成不同的格式後輸出。

### 用 `TextElementEnumerator`
這就和之前那篇一樣了。 

### 結論
優先使用 `string` 提供的方法，包含 `String.Normalize(...)`，兩者都不合用時再考慮 `TextElementEnumerator`，不要一開始就繞遠路。

### 參考
[Char objects and Unicode characters](https://docs.microsoft.com/en-us/dotnet/api/system.string?view=netstandard-2.1#char-objects-and-unicode-characters)  

[UNICODE NORMALIZATION FORMS](https://unicode.org/reports/tr15/)  

[Unicode Normalization 文字標準化](https://blog.sean.taipei/2021/12/unicode)  
