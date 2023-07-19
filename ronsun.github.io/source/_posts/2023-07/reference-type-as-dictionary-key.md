---
title: 參考型別當 Dictionary Key 的注意事項
date: 2023-07-20 00:13:05
categories:
- C#
- .NET
tags:
---

`Dictionary<TKey, TValue>` 是很常用的資料結構，我們可以使用索引子 (Indexer) 透過特定的 Key 來存取資料。 但使用時，我們要特別注意 Key 的型別，使用值型別作為 Key 通常不會有問題，但當使用參考型別作為 Key 時就不一定了。

<!--more-->

### 使用參考型別作為 Key 時的存取
先看下面程式碼：
``` csharp
public class TwoValue<T1, T2>
{
    public TwoValue(T1 v1, T2 v2)
    {
        Value1 = v1;
        Value2 = v2;
    }
    public T1 Value1 { get; }
    public T2 Value2 { get; }
}

// The key is `new TwoValue<int, int>(1, 2)`
var dic = new Dictionary<TwoValue<int, int>, string>();
dic.Add(new TwoValue<int, int>(1, 2), "Hello");

dic.TryGetValue(new TwoValue<int, int>(1, 2), out var v1).Dump(); // false
v1.Dump(); // null
```

從 [Dictionary 的 Add(...) 方法](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs,191) 中可以看到，在新增項目的時候，Key 的部分是以呼叫 `GetHashCode()` 方法所得到的結果為基準產生的。我們也知道參考型別的不同物件，呼叫 GetHashCode 得到的結果會不同，所以新增項目後又以另外一個新建的 `TwoValue` 物件為 Key 去查詢，就會因為新物件的 `GetHashCode()` 回傳和 `Dictionary` 內物件的不同而找不到。

> 關於 `Object.GetHashCode()` 怎麼運作、和物件的實際記憶體位置有什麼關係、和 `Object.Equal(object)` 又有什麼關係，之後再找時間仔細了解。

### 使用 Tuple 作為 Key 時的存取
但是如果使用的是 `Tuple` 就不同了，如下範例：
``` csharp
// The key is `new Tuple<int, int>(1, 2)`
var dic = new Dictionary<Tuple<int, int>, string>();
dic.Add(new Tuple<int, int>(1, 2), "Hello");

dic.TryGetValue(new Tuple<int, int>(1, 2), out var v2).Dump(); // true
v2.Dump(); // "Hello"
```

雖然 `Tuple<T1,T2>` 是參考型別，但因為他[覆寫了 `GetHashCode()` 方法](https://referencesource.microsoft.com/#mscorlib/system/tuple.cs,225) 使得產生的雜湊值其實是源自於它包含的 T1 與 T2 的雜湊值，所以才會造成參考型別的 Key 即使是不同物件也能用來查詢 Dictionary 項目的現象。  

這點其實很重要，如果不知道這些細節，可能會因為這些程式碼執行結果而反向推斷出像是 "`Tuple` 是值型別" 這樣的錯誤認知，或是對參考型別的特性產生困惑與混淆。


### 使用包含參考型別的 Tuple 作為 Key 時的存取
經過上面兩種情境的示範，應該不能猜出這個情境會有什麼結果了，範例如下：
``` csharp
public class TwoValue<T1, T2>
{
    public TwoValue(T1 v1, T2 v2)
    {
        Value1 = v1;
        Value2 = v2;
    }
    public T1 Value1 { get; }
    public T2 Value2 { get; }
}

// The key is `new TwoValue<int, int>(1, 2), 3)`
var dic = new Dictionary<Tuple<TwoValue<int, int>, int>, string>();
dic.Add(new Tuple<TwoValue<int, int>, int>(new TwoValue<int, int>(1, 2), 3), "Hello");

dic.TryGetValue(new Tuple<TwoValue<int, int>, int>(new TwoValue<int, int>(1, 2), 3), out var v3).Dump(); // false
v3.Dump(); // null
```

因為 `Tuple<T1, T2>` 是以其所包含的型別參數們的雜湊值為基礎去產生他自己的雜湊值，所以當然如果型別參數中有參考型別，那整個 `Tuple<T1, T2>` 的雜湊值就可能 (如果他沒有改變 `GetHashCode()` 的行為) 會因為其所包含的成員產生不同雜湊值而最終結果也不同，不能單以型別參數是參考型別而判定他適合做為 `Dictionary` 的 Key。

### 結論
總結來說就是， `Dictionary<TKey, TValue>` 中的 TKey 盡量不要用參考型別，如果要用得要確定他不會產生上述的疑慮。  

其實不難理解，但是很難用文字說明，自己看起來都像是在繞口令...  

### 參考

[Reference Source](https://referencesource.microsoft.com/)