---
title: 不依賴 DDEX provider 使用 Entity Framework 操作 SQLite
date: 2019-10-17 23:33:43
categories:
- C#
- .NET
tags:
---

在之前的文章 {% post_link use-sqlite-with-ef-and-vs2017 %} 使用方式雖然方便, 但是相對的非常依賴工具, 所以這篇用來記錄用盡量少的工具來使用 Entity Framework 操作 SQLite.    

<!--more-->

### 套件與工具安裝
不同於之前, 這次只需要
+ [安裝 NuGet 套件 System.Data.Sqlite](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider#install-systemdatasqlite-nuget-package)  
System.Data.Sqlite 會一併安裝依賴的套件與 Entity Framework 本身.  

### 設定 App.config
App.config 中主要有三個部分要設定, 有些在安裝 System.Data.Sqlite 與 Entity Framework 時會有預設的內容, 預設內容可能需要調整, 可運作的版本如下:  

#### `<providers>` 區塊
這個區塊在 `<entityFramework></entityFramework>` 中, 安裝完 Entity Framework 會自動生成一組範本, 修改後只留必要的.  
``` xml
<providers>
  <provider invariantName="SQLiteProvider" type="System.Data.SQLite.EF6.SQLiteProviderServices, System.Data.SQLite.EF6" />
</providers>
```

#### `<system.data>` 區塊
這個區塊在 `<configuration></configuration>` 中.  
``` xml
<system.data>
  <DbProviderFactories>
    <add name="SQLite Data Provider" invariant="SQLiteProvider" description=".NET Framework Data Provider for SQLite" type="System.Data.SQLite.SQLiteFactory, System.Data.SQLite" />
  </DbProviderFactories>
</system.data>
```

#### `<connectionStrings>` 區塊
這個區塊在 `<configuration></configuration>` 中, 跟 `<system.data>` 同一層.  

``` xml
<connectionStrings>
  <add name="NorthwindConnection" providerName="SQLiteProvider" connectionString="data source=C:\Users\Ron\Desktop\SqliteDemo\SqliteDemo\Northwind.sqlite"/>
</connectionStrings>
```

#### 說明
相較於之前透過工具產生的內容, 這邊手動調整後簡短很多, 要注意的大概就是 providerName, invariant 和 invariantName 要對得起來, connectionString 寫好就好.  

### 新增 Model
以北風資料庫中的 Category 表為例, 新增一個相應的 Model 如下:  
``` csharp
public partial class Category
{
    public long Id { get; set; }
    public string CategoryName { get; set; }
    public string Description { get; set; }
}
```

### 新增 NorthwindContext
繼承 `DbContext` 並覆寫 `OnModelCreating(DbModelBuilder)` 方法, 設定資料表跟 Model 的對應, 並增加相應的 `DbSet<T>`.  

``` csharp
public partial class NorthwindContext : DbContext
{
    public NorthwindContext()
        : base("name=NorthwindConnection")
    {
    }
    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Category>().ToTable("Category");
        base.OnModelCreating(modelBuilder);
    }
    public virtual DbSet<Category> Categories { get; set; }
}
```

> `DbContext` 的部分就我目前理解是 UnitOfWork Pattern 的實作, 而 `DbSet<T>` 則像是 Generic Repository.  

### 寫段程式用用看
``` csharp
class Program
{
    static void Main(string[] args)
    {
        using (var context = new NorthwindContext())
        {
            var categories = context.Categories.ToList();
            foreach (var category in categories)
            {
                Console.WriteLine($"{category.CategoryName}: {category.Description}");
            }
        }
        Console.ReadLine();
    }
}
```

### 結論
用這種方式來優點就是彈性, 減少開發工具的相容性的問題, 另一方面, 實作過程能更清楚每個環節的角色, 即使不完全了解細節也能避免過度依賴工具自動產生程式碼而忘記要去了解內容. 

缺點也是顯而易見的, 資料庫欄位變動後都需要手動維護 Model 和 `NorthwindContext`, 沒有工具幫忙同步, 維護成本高.

總結來說, 用在產品開發上太耗時間有點不實際, 但作為工具限制下的方案或是學習用途還是很適合的.  