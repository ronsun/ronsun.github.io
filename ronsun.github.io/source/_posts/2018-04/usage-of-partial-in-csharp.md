---
title: C# 修飾詞 - partial
date: 2018-04-09 00:15:33
categories:
- C#
- Language Spec
tags: 
- partial
---

C# 的 partial 修飾詞是相對少用的特性, 但在某些時候能起到很關鍵的作用, 所以還是知道一下比較好.  
partial不是關鍵字但是他可以放在在class, struct, interface 以及 void method(...) 的前面作為修飾詞, 並將類型宣告或方法拆分成多個, 而編譯時會將所有區段結合起來, 所以對執行時期沒有影響. 

<!--more-->

### 用法 

#### 部分類別(partial class)
以部分類別為例, 我們可以將一個類別拆成多個部分類別並且分別放在不同的多個檔案中, 如下:

**OrderService.cs**
``` casharp
public partial class OrderService
{
    // something...
}

```

**OrderService2.cs**
``` csharp
public partial class OrderService
{
    // something...
}

```

> 以目前所知patial的用途對於 interface, struct 都相似於 class

#### 部分方法(partial method)
方法宣告在其中一個部分類別中, 並在另外其中一個部分類別中實作已宣告的部分方法.

**OrderService.cs**
``` csharp
public partial class OrderService
{
    //宣告
    partial void CreateOrder();
}
```

**OrderService2.cs**
``` csharp
public partial class OrderService
{
    //實作
    partial void CreateOrder()
    {
        // ...
    }
}
```

---

### 用途
那麼, 什麼情境需要特意將相同類別拆分到不同檔案中呢? 下面依然以部分類別為出發點說明.  

> 部分方法的情境目前還沒遇過, 暫時想不到範例

#### 擴充由工具產生的程式碼
工具產生的程式碼原則上是不允許人為去直接修改內容的, 因為當下次重新產生的時候, 會將修改的部分也覆蓋掉, 所以我們會透過 partial 將工具產生的程式碼與人為擴充的程式碼分開.  
這方面的應用常見的是在擴充 EF(Entify Framework) 自動產生的 Models 上, EF產生的 Models 是基於資料庫的欄位的設計的, 有時候我們需要加上一些欄位方便使用時就能派上用場, 例如:  

Member.cs (工具產生)
``` csharp
public partial class Member
{
    public string FirstName { get; set; }

    public string LasName { get; set; }
}
```

MemberExtend.cs (人為擴充)  
``` csharp
public partial class Member
{
    public string FullName
    {
        get
        {
            return $"{FirstName} {LasName}";
        }
    }
}
```

上面的例子中, EF在更新Models時只會覆蓋 `Member.cs`, 對於擴充部分可以不用擔心被影響.  


#### 類別過大且難以分割
有些類別本身包含大量的內容, 且因為種種因素難以拆分時, 就可以利用partial並將其拆分成不同檔案.  

> 如果可以的話還是重構出更小的單元, 分成多個部分類型不是優先選項.

#### 重構單一類別時
這是運用在之前公司的專案上的, 當時因為類別中的程式碼過多且複雜, 無法一次重構完, 又擔心當下只重構一部分會讓之後要繼續時需要重新花時間再看過一遍, 就將重構後的程式碼拆成另一個檔案, 等之後全部整理完再合併成一個.

---

### 限制
由於部分類型編譯後會被視為一個類型, 其限制的大原則是不能與這個特性矛盾, 這邊大致列出一些.

#### 部分方法
+ 回傳必須為 void, 且 partial 一定要放在 void 前面
+ 不可明確指定存取修飾詞, 隱含為 private
+ 一個方法最多只能在所有部分類別中宣告一次
+ 最多只能有一個以下的實作
+ 可以宣告後不實作, 但不能有實作沒宣告
+ 參數不可有 out 修飾詞
+ 無法明確實作介面方法

> 其餘族繁不及備載, 更多細節參閱 C# 規格書或 MSDN

#### 其他部分類型
所有部分類型的
+ 存取修飾詞不可衝突(但可以只有一個部分類型明確指定修飾詞)
+ 泛型參數的數量、順序與名稱必須完全一致
+ 泛型參數的條件約束不可衝突(可以只有一個部分類型明確指定條件約束)

> 更多細節參閱 C# 規格書或 MSDN

---

### 完整特性
關於partial修飾詞的完整特性(非常多...), 參閱C#規格書, 相關目錄如下(主要在第10章的幾個小節中, 其他章就只是簡介然後說參考第10章).

> 10. 類別  
> 10.1.2 Partial 修飾詞  
> 10.2 部分類型  
> 10.6.8. 部分方法  
> 11. 結構  
> 11.1.2 partial 修飾詞  
> 13. 介面  
> 13.1.2. partial 修飾詞  

---

### 參考與延伸
C#語言規格書  
[MSDN](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods)
[Introduction to Partial Methods](https://www.codeproject.com/Articles/30101/Introduction-to-Partial-Methods)
