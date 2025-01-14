---
title: LINQ 的 Method Syntax 和 Query Syntax 的比較
date: 2025-01-14 23:51:39
categories:
- C#
- .NET
tags:
---

LINQ 語法可以用 Method Syntax 和 Query Syntax 來表示，在應用上我通常會視情境決定如何使用與搭配，但一直沒有整理過選擇的依據。今天剛好有同事問到，就趁這個機會整理一下。

<!--more-->

### Method Syntax
Method Syntax 是 LINQ 中最常用的語法形式，語法範例如下：
``` csharp
var query = customers
    // GroupJoin + SelectMany for left join
    .GroupJoin(
        orders,
        customer => customer.CustomerId,
        order => order.CustomerId,
        (customer, customerOrders) => new { Customer = customer, Orders = customerOrders })
    .SelectMany(
        x => x.Orders.DefaultIfEmpty(),
        (customerWithOrders, order) => new { customerWithOrders.Customer, Order = order })
    .WhereIf(r => r.Order.TotalAmount > totalAmount, totalAmount.HasValue)
    .GroupBy(
        r => r.Customer.Name,
        (key, group) => new
        {
            CustomerName = key,
            TotalOrderAmount = group.Sum(r => r.Order?.TotalAmount ?? 0),
            Orders = group.Select(r => r.Order).ToList()
        });
```

#### 適用情境
+ 需要搭配自定義的 LINQ 擴充方法時：  
  例如上述範例中的 `WhereIf()`，或是 `Paging(condition)`... 等各種自定義的場景。
+ 需要使用 Method Syntax 才有提供的方法時：  
  例如 `AsNoTracking()`、`Include()`、`SelectMany()`、`Take()`... 等。
+ 不需要處理複雜的 JOIN 操作時：  
  雖然 Method Syntax 支援 JOIN，但相關方法參數複雜，尤其是處理 LEFT JOIN 時語法會變得很複雜。
+ ORM 的 Entity Model 提供了關聯屬性時：  
  如果可以透過 ORM（如 Entity Framework）的導覽屬性（Navigation Properties）直接取代 JOIN，Method Syntax 的語法會更簡潔。

### Query Syntax
Query Syntax 語法更接近於傳統的 SQL 查詢語法，這種語法形式更直觀。語法範例如下：

``` csharp
var query = 
    from customer in customers
    // join + into + DefaultIfEmpty for Left join
    join order in orders on customer.CustomerId equals order.CustomerId into customerOrders
    from order in customerOrders.DefaultIfEmpty()
    where order.TotalAmount > totalAmount
    group new { Customer = customer, Order = order } by customer.Name into grouped
    select new
    {
        CustomerName = grouped.Key,
        TotalOrderAmount = grouped.Sum(r => r.Order?.TotalAmount ?? 0),
        Orders = grouped.Select(r => r.Order).ToList()
    };
```

#### 適用情境
+ 不需要搭配自定義 LINQ 擴充方法時：  
  Query Syntax 本身不支援自定義擴充方法，若需要此功能則可能需要改用 Method Syntax，或和 Method Syntax 混合使用。
+ 需要處理複雜的 JOIN 操作時：  
  Query Syntax 尤其在處理 JOIN（尤其是 LEFT JOIN）的語法上，比 Method Syntax 更簡潔直觀。
+ 無法以 ORM 的 Entity Model 關聯屬性處理關係時：  
  當需要依據多個非主鍵/外鍵欄位進行關聯操作時（如複雜的條件式關聯），Query Syntax 提供了更靈活的方式來表達這些查詢邏輯。

### 結論
兩種方法各有優缺與限制，且可以分開實作後再組合後混合使用，釐清各自的特性與優缺後適當選擇能讓程式碼更易懂且乾淨。

### 參考
ChatGPT
