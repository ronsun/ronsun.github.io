---
title: Entity Framework Core 預熱
date: 2024-05-12 00:19:21
categories:
- C#
- Packages
tags:
---

在使用 Entity Framework Core (EF Core) 進行資料存取操作時，初次查詢時會有耗時較長的現象。這種延遲主要是因為 EF Core 在第一次執行查詢時需要進行一系列的初始化步驟。這些步驟造成的延遲可能會對效能敏感的應用程式造成不利的影響。例如在 WebAPI 服務中，如果初始化花費太多時間可能造成連線超時。而且在效能監測時，這種初始化所造成的延遲可能會產生極端的執行時間資料影響到效能的解讀。

為了解決這個問題，通常會在應用程式啟動階段加入一些預熱的操作，以提前完成一些必要的初始化步驟。這篇文章會介紹兩種預熱解法，**要注意的是這兩個解法不是針對所有的初始化步驟，目的只是要達到觸發 `DbContext.OnModelCreating()` 的效果來降低首次查詢的延遲時間**。

<!--more-->

### 解法一： `dbContext.Set<T>`

以 ASP.NET Core 來說，在 `Startup.cs` 中的 `Configure` 方法中注入 `DbContext` 來達到效果是最常見的作法，如下：

``` csharp
public void Configure(IApplicationBuilder app, DbContext dbContext)
{
    // Warmup operation, trigger DbContext.OnModelCreating() to reduce the initialization delay on first query.
    dbContext.Set<Blog>().FirstOrDefault();
}
```

這是最簡單的解法，但特定的 Model 進而產生額外的依賴這個缺點會帶來幾個風險和限制：
+ 該 Model 被移除時看似無關的預熱功能也需要一併修改而使用其他 Model。
+ 會需要特別寫註解說明目的，否則後期維護時很可能被誤當成多餘的程式碼而刪除，最麻煩的是刪除後短期內可能還不會產生明顯的問題。
+ 難以通用化，例如公司內部如果有通用套件時，會因為依賴特定資料庫對應的 Model 而無法將預熱功能加入其中。

### 解法二： `ExecuteSqlRaw()`
為了避免解法一所產生的副作用，可以用執行一個和資料表無關的查詢達到效果，如下：

``` csharp
public void Configure(IApplicationBuilder app, DbContext dbContext)
{
    // Warmup operation, trigger DbContext.OnModelCreating() to reduce the initialization delay on first query.
    dbContext.Database.ExecuteSqlRaw("SELECT 1 FROM DUAL");
}
```

這個解法解決了依賴特定的 Model，這個優點帶來了很大的通用性。以我遇到的情境來說，能輕易的將預熱功能抽離到內部套件中提供跨部門同事使用，如下範例：

``` csharp
// In internal library.
public namespace MyCompany.EFCoreEnhancement
{
    public static class DbContextExtension
    {
        public static void Warmup(this DbContext context)
        {
            context.Database.ExecuteSqlRaw("SELECT 1 FROM DUAL");
        }
    }
}
```

``` csharp
// In application using the internal library.
public void Configure(IApplicationBuilder app, DbContext dbContext)
{
    dbContext.Warmup();
}
```

### 結論
一般來說除非已有現成 API 可用，否則不建議一開始就做預熱功能。如果要做則是視使用情境，單一專案或存取單一資料庫的情境解法一加註解就很夠用了。解法二則是基於最大化通用性的考量設計的，這個方案在把預熱功能抽離到通用套件時才有明顯的優勢。
