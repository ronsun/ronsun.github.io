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

### 題外話： 傳值 (pass by value) 與傳址 (pass by reference) 的差異
這個題目常和實值型別與參考型別一起被問到，常見的誤解是以為參考型別作為參數傳遞時就是傳址。實際上不論型別是實值還是參考，C# 預設都是傳值，差別只在於複製的內容是什麼。

#### 傳值與傳址各自傳的是什麼
+ 傳值 (pass by value)：複製變數的內容傳入方法。
  - 實值型別變數的內容是資料本身，傳入的是資料的副本。
  - 參考型別變數的內容是指向堆積中物件的位址，傳入的是位址的副本，兩邊仍指向同一個物件。
+ 傳址 (pass by reference)：被呼叫端的參數是一個 reference variable，本身沒有值，refer to 呼叫端傳入的變數（即其 referent）；對該參數的讀寫等同對 referent 的讀寫，實值型別與參考型別皆適用。
  - C# 中以 `ref`、`out`、`ref readonly`、`in` 關鍵字標示傳址。

#### 對外部變數的影響差異
+ 參考型別以傳值傳入時，方法內透過參數修改物件成員會反映到外部，因為兩邊指向同一個物件；但若在方法內重新指派 (reassign) 參數到另一個物件，外部變數不受影響，因為改動的只是副本中存的位址。
+ 以傳址傳入時，方法內的重新指派會直接反映到外部變數，因為對參數的寫入即為對其 referent（即呼叫端變數）的寫入。
+ 換句話說，傳值複製的是變數的內容（實值型別是值本身，參考型別是物件的位址）；傳址則讓被呼叫端的參數成為一個 reference variable，referent 為呼叫端傳入的變數，兩者層次不同。

#### 以程式碼與圖示對照兩者
為了同時涵蓋實值與參考型別，先看一段範例程式碼：

```csharp
class Point { public int X; }

void Caller()
{
    int n = 5;
    Point p = new Point { X = 5 };

    // 傳入 n 的內容 5
    // 傳入 p 的內容 0x1000，0x1000 是 Point 物件所在的位置
    ByValue(n, p);

    // 傳入 n 的位置 0x2000，0x2000 位置的內容是 5
    // 傳入 p 的位置 0x2004，0x2004 位置的內容是 0x1000，0x1000 是 Point 物件所在的位置
    ByRef(ref n, ref p);
}

void ByValue(int m, Point obj)
{
    // 寫入 10 到 m 的位置 0x3000，呼叫端 n 的位置 0x2000 內容仍是 5
    m = 10;
    // 透過 obj 的內容 0x1000 定位到 Point 物件改 X，呼叫端 p 的內容也是 0x1000，故可見變更
    obj.X = 99;
    // 寫入新物件位置到 obj 的位置 0x3004，呼叫端 p 的位置 0x2004 內容仍是 0x1000
    obj = new Point();
}

void ByRef(ref int m, ref Point obj)
{
    // m 是 reference variable，referent 為呼叫端 n（位置 0x2000），寫入 10 即寫入 n，0x2000 內容由 5 變 10
    m = 10;
    // obj 是 reference variable，referent 為呼叫端 p（位置 0x2004），寫入新物件位置即寫入 p，0x2004 內容由 0x1000 變新物件位置
    obj = new Point();
}
```

圖中 stack 上每個變數畫成一個 cell（左側為變數名，cell 內為其值或位址，右側為該 cell 的記憶體位址）。Heap 上的物件單獨繪出，當 stack cell 內容為位址時用箭頭指向對應的 heap 物件。

ByValue：呼叫端與被呼叫端各自擁有獨立的儲存位置

```text
        Stack                                       Heap

        ┌──────────┐
    n → │    5     │ 0x2000
        ├──────────┤
    p → │  0x1000  │ 0x2004 ──┐
        ├──────────┤          │              ┌──────────┐
    m → │    5     │ 0x3000   ├─────────────►│  X = 5   │ 0x1000
        ├──────────┤          │              └──────────┘
  obj → │  0x1000  │ 0x3004 ──┘
        └──────────┘

  函式內執行：
    m = 10              → 寫入 0x3000，0x2000 內容仍是 5
    obj.X = 99          → 透過 0x1000 改物件，0x2004 內容仍是 0x1000，呼叫端可見變更
    obj = new Point()   → 寫入 0x3004，0x2004 內容仍是 0x1000
```

ByRef：被呼叫端的參數是 reference variable，本身沒有值，referent 為呼叫端對應的變數

```text
        Stack                                       Heap

        ┌──────────┐ ◄── m   (referent: n)
    n → │    5     │ 0x2000
        ├──────────┤ ◄── obj (referent: p)
    p → │  0x1000  │ 0x2004 ──────────────────────►┌──────────┐
        └──────────┘                               │  X = 5   │ 0x1000
                                                   └──────────┘

  函式內執行：
    m = 10              → 寫入 m 即寫入 referent n（位置 0x2000），內容由 5 變 10
    obj = new Point()   → 寫入 obj 即寫入 referent p（位置 0x2004），內容由 0x1000 變新物件位置
```

> 可能有人會和我有相同的疑惑：範例中 `obj = new Point()` 不該是改寫參數所存的位址嗎？這是誤以為 `ref` 參數是存著別人位址的獨立變數。[C# 文件對 reference parameter 的描述](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#reference-parameters) 為：
> 
> A parameter that's passed by reference is a *reference variable*. It doesn't have its own value. Instead, it refers to a different variable called its *referent*.
> 
> `obj` 沒有自己的值，它的 referent 就是呼叫端的 `p`；對 `obj` 寫入即為對 `p` 寫入。

### 結論
C# 中實值型別與參考型別的差異，是面試常見題目。雖然許多人可以簡略回答出基本差異，但深入理解記憶體配置、生命週期與效能影響，有助於在實務與面試中展現更紮實的底層知識。

### 參考
ChatGPT  

Claude Code  

[The System.ValueType type](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/types#832-the-systemvaluetype-type)  

[ref (C# Reference)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/ref)  

[Reference parameters (C# reference - Method parameters)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters#reference-parameters)