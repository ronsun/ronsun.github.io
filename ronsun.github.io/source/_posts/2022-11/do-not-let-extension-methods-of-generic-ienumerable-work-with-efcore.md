---
title: 避免在需要使用 EF Core 的環境擴充 IEnumerable< T >
date: 2022-11-15 23:32:38
categories:
- C#
tags:
---

擴充方法是很常用的技巧, 之前在使用 Entity Framework Core 的時候, 擴充了一個 `IQueryable<T>` 的擴充方法 `WhereIf()`, 後來想說這個方法也適用於其他衍生自 `IEnumerable<T>` 的型別, 且 `IQueryable<T>` 繼承了 `IEnumerable<T>`, 所以把 `WhereIf()` 方法改成 `IEnumerable<T>` 的擴充方法以求更廣的適用範圍, 沒想到一切都不一樣了, 要是當時沒及時發現就引爆了一個效能核彈了.  

> 雖然標題是寫 Entity Framework Core, 但 `IQueryable<T>` 本來就是設計給"資料查詢"的情境來說, 其他 ORM 八九不離十會遇到一樣的現象.  

<!--more-->

### 問題描述

以下面的程式碼為範例來說明:  

``` csharp
var adults = _dbContext.Set<Person>() // DbSet<Person>
    .Where(r => r.Age >= 18)          // Append condition, IQueryable<Person>
    .ToList();                        // Query
```

我們知道在 `ToList()` 被呼叫前的行為都只是在組合查詢條件, 不會真正去查資料, 基於這個前提如一開始描述擴充一個 `IEnumerable<T>` 的擴充方法 `WhereIf(...)` 來方便使用, 如下:  

``` csharp
public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereIf<T>(this IEnumerable<T> query, Func<T, bool> predicate, bool shouldAppendWhere)
    {
        if (shouldAppendWhere)
        {
            return query.Where(predicate);
        }

        return query;
    }
}

var adults = _dbContext.Set<Person>()
    .WhereIf(r => r.Gender == condition.Gender.Value, condition.Gender.HasValue) // Optional filter
    .Where(r => r.Age >= 18)
    .ToList();
```

雖然執行結果沒錯, 但這樣做會有個很嚴重的問題, 以上面的例子來說, 當 `WhereIf(...)` 被呼叫時會從資料來源查詢資料, 以 Oracle 來說就是執行了 `SELECT * FROM PERSON` 的查詢將 Person 資料表的 **所有資料** 搜尋出來後才在應用程式中做後續的篩選和處理.  

### 為什麼會有這個現象?
其實從 `IQeueryable<T>` 和 `IEnumerable<T>` 的用途與差別大概就能推測出會有這樣的結果了, 不過出於好奇還是稍微實驗一下看會不會有更明確的答案.  

實驗程式碼如下: 
``` csharp
public static IEnumerable<T> WhereIf<T>(this IEnumerable<T> query, Func<T, bool> predicate, bool shouldAppendWhere)
{
    if (shouldAppendWhere)
    {
        return query.Where(predicate);
    }

    return query;
}

public class MyQueryable<T> : IQueryable<T>
{
    public Type ElementType => typeof(T);

    public Expression Expression => default;

    public IQueryProvider Provider => default;

    public IEnumerator<T> GetEnumerator()
    {
        throw new NotImplementedException("a");
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        throw new NotImplementedException("b");
    }
}

IEnumerable<int> query1 = new MyQueryable<int>().WhereIf(r => r > 10, true);
IEnumerable<int> query2 = new MyQueryable<int>().WhereIf(r => r > 10, false);

"Correct here.".Dump();
// NotImplementedException("a")
query1.Dump();

// NotImplementedException("b")
query2.Dump();

```

經過實驗, 可以看到有加其他條件和直接轉型的情境會分別呼叫到兩個不同的 `GetEnumerator()` 方法, 翻了一下 Entity Framework Core 的原始碼可以發現 [InternalDbSet<T> 中的兩個 GetEnumerator() 方法的實作](https://github.com/dotnet/efcore/blob/main/src/EFCore/Internal/InternalDbSet.cs#L459) 最後都會呼叫到 `CreateEntityQueryable()` 方法且回傳一個 `EntityQueryable<T>` 型別的物件, 接著從 [EntityQueryable<T> 中相關的方法](https://github.com/dotnet/efcore/blob/c771d25cd11a27d268cc7d4fdeeba3c4c4203386/src/EFCore/Query/Internal/EntityQueryable%60.cs#L86) 的實作大概就可以**推測**到會去執行查詢了.  

> **也就是說, 只要觸發 GetEnumerator() 方法的呼叫, 就會引發資料查詢.**  

> 這部分這樣推測是比較粗糙的作法, 嚴格來說應該是要找到真的去執行的程式碼才能證實, 但因為實測已經知道結果了, 加上懶得在家建立完整的環境追蹤, 所以就沒堅持要找到最底層.  


### 解決方案

#### 方案一 : 同時擴充 `IQueryable<T>` 和 `IEnumerable<T>`

如下範例:  
``` csharp
public static class QueryableExtensions
{
    public static IQueryable<T> WhereIf<T>(this IQueryable<T> query, Expression<Func<T, bool>> predicate, bool shouldAppendWhere)
    {
        if (shouldAppendWhere)
        {
            return query.Where(predicate);
        }

        return query;
    }
}

public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereIf<T>(this IEnumerable<T> query, Func<T, bool> predicate, bool shouldAppendWhere)
    {
        if (shouldAppendWhere)
        {
            return query.Where(predicate);
        }

        return query;
    }
}

var adults = _dbContext.Set<Person>()
    .WhereIf(r => r.Gender == condition.Gender.Value, condition.Gender.HasValue) // Optional filter
    .Where(r => r.Age >= 18)
    .ToList();
```

雖然同時擴充 `IQueryable<T>` 和 `IEnumerable<T>` 可以解決問題, 但這有幾個缺點:  
1. 因為 `DbSet<T>` 實作了 `IQueryable<T>`, 而 `IQueryable<T>` 繼承了 `IEnumerable<T>`, 所以編譯時優先使用 `IQueryable<T>` 的擴充方法, 雖然符合預期, 但完全依賴於編譯時的優先順序, 非常隱晦.  
2. 考慮到搭配選擇性引數 (Optional Arguments) 使用時, 很容易在無意間因為一點小改動而導致使用了 `IEnumerable<T>` 的擴充方法而沒發現.  
3. 萬一維護過程將 `IQueryable<T>` 的擴充方法移除或重新命名, 呼叫端還是可以正確編譯的, 但會變成呼叫了 `IEnumerable<T>` 的擴充方法, 而且很難意識到這個問題.  

> **這是個有用且支援範圍廣但需要謹慎維護的解決方案, 適合非常嚴謹的團隊.**  

> 事實上, dotnet runtime 就是同時做了 [`IEnumerable<T>` 版本的 `Where()` 擴充方法](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Linq.Queryable/src/System/Linq/Queryable.cs#L47) 和 [`IQueryable<T>` 版本的 `Where()` 擴充方法](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/libraries/System.Linq/src/System/Linq/Where.cs#L12)


#### 方案二 : 只擴充 `IQueryable<T>`

如下範例:  
``` csharp
public static class QueryableExtensions
{
    public static IQueryable<T> WhereIf<T>(this IQueryable<T> query, Expression<Func<T, bool>> predicate, bool shouldAppendWhere)
    {
        if (shouldAppendWhere)
        {
            return query.Where(predicate);
        }

        return query;
    }
}

var adults = _dbContext.Set<Person>()
    .WhereIf(r => r.Gender == condition.Gender.Value, condition.Gender.HasValue) // Optional filter
    .Where(r => r.Age >= 18)
    .ToList();
```

這樣做不容易出錯, 但也有一個缺點 - 對於衍生自 `IEnumerable<T>` 的型別不友善.  

以下面範例來說, 需要透過 `AsQueryable()` 方法轉型後才能使用.  
``` csharp
var dataSource = new List<Person>();
var adults = dataSource
    .AsQueryable()
    .WhereIf(r => r.Gender == condition.Gender.Value, condition.Gender.HasValue) // Optional filter
    .Where(r => r.Age >= 18)
    .ToList();
```

> **是個支援範圍比較窄, 但是維護風險較低的做法, 適合無法保證謹慎維護的專案.**  

### 結論
整體來說, 如果可能會被任何需要 ORM 的用戶端程式呼叫到, 優先避免做 `IEnumerable<T>` 的擴充方法, 真的需要的話要考慮如果會被 `IQueryable<T>` 的衍生類別呼叫到就必須連同 `IQueryable<T>` 的擴充方法一起做.  

還好在確認 ORM 產生的 SQL Script 內容時就即時發現, 不然等到上 production 才爆發就真的很難找到這麼隱晦的問題來源了.  