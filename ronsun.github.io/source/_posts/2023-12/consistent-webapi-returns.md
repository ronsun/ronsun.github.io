---
title: 設計 WebAPI 通用回傳模板
date: 2023-12-16 23:31:15
categories:
- C#
tags:
---

在我以往的工作經驗中，處理過不少 WebAPI 的維護工作，普遍面臨的一個問題是回傳資料的不一致性。舉例來說，常見的幾種回傳格式包括：
1. 純值： `true`、`100` 等，會出現在回傳單一值的 API 中。
2. 物件： `{"name": "Ron", "gender": "Male"}`，會出現在回傳物件的 API 中。
3. 物件含狀態： `{"status": "1001", "name": "Ron", "gender": "Male"}`，會出現在呼叫端需要知道更多狀態細節時。
4. 其他變種：除了資料外還有各種狀態、錯誤、訊息等欄位，但格式與名稱不統一。

這個現象造成的問題就是明明是同一個站台對外提供服務，回傳的狀態訊息卻每個 API 都不一樣，不僅對使用者不友善，也會讓接手維護的人無所適從。

本篇文章會介紹並分析的幾種解決方案，所有方案都有高一致性與關注點分離的共同優點，但也有些不同的缺點。

<!--more-->

### 用 Header 表示狀態
這是個很常見的做法，用 Header 表示狀態，而回傳的 Body 中則只包含資料本身。  

**優點**  
+ 把狀態和資料徹底區分開，具有高度彈性和相容性，尤其適合於需要回傳非字串類型資料 (如串流、multipart/form-data) 的情境。

**缺點**  
+ 管理不便
  - 需要較多文件輔助。
  - 需要適當設計來集中來收納各種 Header 的常數，避免到處散落的 Magic String。
+ 較不直覺，開發者和使用者都需要認知到 Header 中有包含狀態訊息。

### 用基底類別收納狀態相關欄位
``` csharp
public class ResponseModelBase
{
    public string Status { get; set; }
    
    public string Message { get; set; }
}

public class Person : ResponseModelBase
{
    public string Name { get; set; }
    public Gender Gender { get; set; }
}
```
如上面範例，使用 `ResponseModelBase` 做為基底類別來收納狀態類的資訊，他會被所有 API 的回傳類別繼承，他與使用 Header 的解決方案有著相似的優點並解決了一些 Header 的缺點，但也產生不同面相的缺點。  

**優點**  
+ 管理稍微容易，只需要提到繼承 `ResponseModelBase` 即可。
+ 直覺，狀態就在 Body 中。

**缺點**  
+ 只支援回傳字串的情境。
+ 如果衍生類別 (如 Person) 與基底類別 (如 ResponseModelBase) 有同名欄位，會造成命名上的困擾、缺陷或混淆。 由於無法預測未來的規格變化，這種風險難以避免。 (基底類別的欄位用很特殊的命名不算個解法)。

### 繼承通用泛型類別
``` csharp
public class ResponseModel<T>
{
    public string Status { get; set; }

    public string Message { get; set; }

    public T Data { get; set; }
}

public class Person
{
    public string Name { get; set; }
    public Gender Gender { get; set; }
}
```

如上範例，借用以包含取代繼承的概念，設計泛型類別取代基底類別，而 API 回傳的型別則會變成 `ResponseModel<Person>`。

**優點**  
+ 管理稍微容易，只需要提到 `ResponseModel<T>` 即可。
+ 直覺，狀態就在 Body 中。
+ 基礎類別 (如 Person) 不用擔心和 `ResponseModel<T>` 中的欄位同名。

**缺點**  
+ 只支援回傳字串的情境。
+ 比起繼承，泛型型別在使用時的程式碼比較冗長。

### 結論
這幾種方案各有利弊，但至少都能解決文章開頭提到的一致性、維護性和使用者友好度的問題。總結建議如下：
1. 當預期會回傳非字串型態的 Body 時，選擇 Header 解決方案，但需要注意管理的複雜性。  
3. 當預期只會回傳字串 Body 時，選擇泛型解決方案。
2. 不要選擇繼承通用類別的解法，提到他只是要記錄他和泛型型別設計上的差異。  

### 參考  
開發經驗。  