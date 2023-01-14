---
title: 跨組件反射的注意事項
date: 2020-08-02 01:29:09
categories:
- C#
- .NET
tags:
---

問題的起源在於某次搬遷程式碼的時候, 雖然看起來只是將一個方法移動到別的專案, 卻不小心讓一個有使用反射的方法在執行階段拋錯, 雖然查明後發現不是太刁鑽的問題, 但卻是容易不小心出錯的, 所以稍微紀錄一下.  

<!--more-->

### 問題點
當時的情況是有兩個專案, 假設是 `LogicLayer` 和 `Shared`, 要將一個 `Create(string typeName)` 方法從原本的 `LogicLayer` 專案搬到 `Shared` 中, 讓這個共用的工具方法能建立在正確的專案上, 方法如下:  

``` csharp
// T must be base type of typeName
public static T Create<T>(string typeName)
{
    var type = Type.GetType(typeName);
    return (T)Activator.CreateInstance(type);
}
```

呼叫端則是這樣呼叫的: 

``` csharp
var typeName = "LogicLayer.DerivedModel";
var type = Create<BaseModel>(typeName);
```

> `Create<T>(string typeName)` 方法沒有這麼單純, 範例只是為了描述問題, 而 "LogicLayer.DerivedModel" 是存在資料庫中的資料, 被取出來後做為引數代入 `Create<T>(string typeName)` 方法中.  

當時覺得就只是將方法原封不動的搬去 `Shared` 專案中, 編譯和單元測試都有過也就不應該出什麼問題, 沒想卻在執行階段卻出現 Exception.  

### 原因與解決方式
主要是因為 `typeName` 並不是目前組件 `Shared` 中的類別, 所以 `Type.GetType(typeName)` 會回傳 `null`, 導致接下來的 NullReferenceException.  

由於這個 `Create<T>(string typeName)` 方法必須在 `Shared` 專案才合理, 不能因為這樣不搬 ; 而如果使用完整的組件限定名稱 (assembly-qualified name), 像這個樣子 `LogicLayer.DerivedModel, LogicLayer, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null` 也不適當, 因為這個值是存在資料庫中的, 在相容舊資料的限制下不能這樣做.  

最後是把 `Create<T>(string typeName)` 改成 `Create<T>(Type type)`, 由呼叫端取得 `Type` 後再傳入, 由於目前系統的前提是 typeName 必定屬於 `LogicLayer` 專案中的類別, 所以這樣做是合理的.  

### 結論
其實是低級錯誤, 因為這個方法其實是有做單元測試的, 但是測資 (`typeName`) 用了完整的組件限定名稱, 跟資料庫的真實資料格式不一樣, 所以測了也是白測才會拖到執行階段才發現錯誤.  

另外題外話就是, 把類別名稱 (type name) 存在資料庫後再拿出來反射, 就我目前的觀點是很不好的行為, 因為這樣子會讓資料庫的資料本身跟應用程式的語言特性產生依賴, 別說應用程式換語言的時候這個資料就不能用了 (或是需要經過轉換才能用) , 就算只要類別改名都能讓舊資料直接造成在執行階段出錯, 變成類別的命名與資料庫資料這兩個八竿子打不著的事情產生依賴, 提高維護的風險與成本.  

> 不只是類別名稱與反射, 應該要避免跟程式語言或框架執行有關的資料進資料庫進而產生依賴, 就算特殊情境也應該特別拿出來專門討論後面的維護風險以及是否有適合的替代方案.  
