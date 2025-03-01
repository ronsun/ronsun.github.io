---
title: 從面試角度看 C# 的實值型別與參考型別
date: 2025-03-02 01:03:40
categories:
- C#
- Language Spec
tags:
---

C# 的實值型別與參考型別雖然是入門主題，但實際上可以深入探討，涉及的範疇也相當廣泛。本文將從面試的角度切入，逐步深入說明相關差異與應用情境。

<!--more-->

### 實值型別與參考型別的差異
這個問題可以從幾個面向探討。  

#### 賦值 (assignment) 與比較 (equality) 時的差異
+ 實值型別在賦值時會完整複製實例 (instance) 的內容，比較時也是比較完整的內容。
+ 參考型別在賦值時複製的是記憶體位址，比較時預設比較的是位址（除非有自行覆寫 `Equals` 或其他比較運算子來改變預設行為）。  

因此，實值型別變數所佔的記憶體大小取決於其內容本身；而參考型別變數則為固定大小，僅儲存參考位址。

#### 記憶體配置的差異
+ 實值型別**通常**都是儲存在堆疊 (stack) 中；參考型別則**一定**儲存在託管堆積 (managed heap) 中。
  - 當實值型別進行裝箱 (boxing) 時，其資料會被複製到堆積；拆箱 (unboxing) 時則會從堆積複製回堆疊。
    * 裝箱與拆箱會產生額外的執行成本，且裝箱會造成資料被複製到堆積中，進而增加 GC 的負擔。
  - 實值型別如果是參考型別的成員，其記憶體配置也會隨同參考型別一起存在堆積中。
+ 這裡的堆疊與堆積並不是資料結構上所說的 Stack 與 Heap，而是兩種不同的記憶體管理方式的名字，名稱只是借用了相關概念，而不是用相應的資料結構實作。

#### 生命週期的差異
+ 儲存在堆疊上的實值型別，其生命週期與所在作用域相同，而作用域通常是由 `{}` 括起來的程式區塊。
+ 儲存在堆積上的參考型別與實值型別，則由 .NET 的垃圾回收機制（GC）負責管理其生命週期。

#### 堆疊與堆積的效能差異
+ 儲存在堆疊中的實值型別，其生命週期極短，初始化時 push 而離開作用域就 pop，不受 GC 管理，因此不會增加 GC 壓力。
+ 堆積中的資料需要等待 GC 機制來回收，所以物件生命週期可能很長，亦即會佔用記憶體較長的時間。

### 如何分辨實值型別和參考型別
+ 列舉 (enum) 和結構 (struct) 都是實值型別，除此之外的所有型別都是參考型別。

> 參考 [The System.ValueType type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/types#832-the-systemvaluetype-type) 更嚴格深入來說：  
> 在 C# 中，所有使用 `struct` 或 `enum` 宣告的型別，背後在 IL 層級都會繼承自 `System.ValueType`，而 `System.ValueType` 是 .NET 中判斷某型別是否為實值型別的依據。雖然在 C# 原始碼中不需要手動繼承 `ValueType`，但這個繼承關係會由編譯器自動加上。
>
> 舉例來說，`System.Int32` 結構反組譯後可看到：  
> `.class public sequential ansi sealed serializable beforefieldinit System.Int32 extends [System.Runtime]System.ValueType`
>

### 設計型別時如何決定設計成實值型別或參考型別
適合使用實值型別的情境：
+ 小型資料：  
  由於實值型別賦值時是完整複製內容，因此不適合承載大型資料，一個實值型別的物件應該小於 16 bytes。

適合使用參考型別的情境：  
+ 大型物件：  
  大型物件設計成參考型別在重用時可以址傳遞參考而不用複製完整內容，降低記憶體使用。
+ 需要繼承時：  
  實值型別無法被繼承。

總結來說，實值型別適合頻繁操作的小型資料、臨時資料以及對效能/記憶體要求極高的情境；而參考型別適合較複雜的大型資料結構、需要共享或長期存在的物件。

### 結論
C# 中實值型別與參考型別的差異，是面試常見題目。雖然許多人可以簡略回答出基本差異，但深入理解記憶體配置、生命週期與效能影響，有助於在實務與面試中展現更紮實的底層知識。

### 參考
ChatGPT

[The System.ValueType type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/types#832-the-systemvaluetype-type)