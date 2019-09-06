---
title: 在 vs2017 中使用 Entity Framework 操作 SQLite
date: 2019-09-07 01:01:57
categories:
- C#
- .NET
tags:
---

有時候做一些實驗的時候會需要用到資料庫, 但是為了實驗特別架一個 SQL Server 的環境實在是很麻煩, 電腦重灌或是虛擬機重裝的時候重建也麻煩, 所以就把腦筋動到 SQLite 上了, 不過操作上沒有想像中的簡單呢.  

因為是為了建構實驗用的資料庫, 不希望花太多時間, 希望能盡可能方便的將資料存取層建起來, 所以採用資料庫優先 (database first) 的方式來建立 ADO.NET Entity Data Model (*.edmx).   

<!--more-->

### 套件與工具安裝
System.Data.SQLite DDEX provider 不支援 Visual Studio 2017, 所以需要借助其他套件的幫忙, [SqlCeToolbox](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider) 提供套件支援與安裝教學, 內容很完整, 所以這邊只稍微列一下步驟.  

+ [安裝 SqlCeToolbox](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider#install-latest-toolbox)

+ [安裝 SQLite 到 GAC](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider#install-sqlite-in-gac)  
這邊要注意只能安裝特定版本才行, 特定版本會有下面的標註 (粗體放大應該不難找), 我是安裝 **Setups for 32-bit Windows (.NET Framework 4.6)** 這個版本.  
    > This is the only setup package that is capable of installing the design-time components for Visual Studio 2015. 

+ [安裝 NuGet 套件 System.Data.Sqlite](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider#install-systemdatasqlite-nuget-package)  
System.Data.Sqlite 會一併安裝依賴的套件與 Entity Framework 本身.  

### 新增 ADO.NET Entity Data Model (*.edmx)

{% asset_img edmx_1.png %}

{% asset_img edmx_2.png %}

{% asset_img edmx_3.png %}

{% asset_img edmx_4.png %}  

{% asset_img edmx_5.png %}

這些步驟有些地方要稍微注意一下
+ **NorthwindContext** 會成為 App.config 中 `connectionStrings` 這一段的預設名字, 如下:  
    **App.config**
    ``` xml
    <connectionStrings>
        <add name="NorthwindContext" connectionString="metadata=res://*/DataAccess.Northwind.csdl|res://*/DataAccess.Northwind.ssdl|res://*/DataAccess.Northwind.msl;provider=System.Data.SQLite.EF6;provider connection string=&quot;data source=C:\Users\Ron\Desktop\SqliteDemo\SqliteDemo\Northwind.sqlite;version=3&quot;" providerName="System.Data.EntityClient" />
    </connectionStrings>
    ```

    **Northwind.Context.cs**
    ``` csharp
    public partial class NorthwindContext : DbContext
    {
        public NorthwindContext()
            : base("name=NorthwindContext")
        {
        }
        
        // skip...
    }
    ```
+ Model Namespace 一欄的值 **"Northwind"** 會在 Northwind.edmx 中用來作為命名空間, 取個合理的名字就好.  

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

沒意外的話, 會拋出 Exception, 預設在 App.config 中填入的設定是需要一些調整的.  

### 調整 App.config
#### 調整
調整一下 App.config 中,  
**DbProviderFactories**
``` xml
<DbProviderFactories>
    <remove invariant="System.Data.SQLite.EF6" />
    <add name="SQLite Data Provider" invariant="System.Data.SQLite.EF6" description=".NET Framework Data Provider for SQLite" type="System.Data.SQLite.SQLiteFactory, System.Data.SQLite" />
</DbProviderFactories>
```

#### App.config 相關內容
節錄:
``` xml
<entityFramework>
    <defaultConnectionFactory type="System.Data.Entity.Infrastructure.LocalDbConnectionFactory, EntityFramework">
        <parameters>
            <parameter value="mssqllocaldb" />
        </parameters>
    </defaultConnectionFactory>
    <providers>
        <provider invariantName="System.Data.SqlClient" type="System.Data.Entity.SqlServer.SqlProviderServices, EntityFramework.SqlServer" />
        <provider invariantName="System.Data.SQLite.EF6" type="System.Data.SQLite.EF6.SQLiteProviderServices, System.Data.SQLite.EF6" />
    </providers>
</entityFramework>
<system.data>
    <DbProviderFactories>
        <remove invariant="System.Data.SQLite.EF6" />
        <add name="SQLite Data Provider" invariant="System.Data.SQLite.EF6" description=".NET Framework Data Provider for SQLite" type="System.Data.SQLite.SQLiteFactory, System.Data.SQLite" />
    </DbProviderFactories>
</system.data>
<connectionStrings>
    <add name="NorthwindContext" connectionString="metadata=res://*/DataAccess.Northwind.csdl|res://*/DataAccess.Northwind.ssdl|res://*/DataAccess.Northwind.msl;provider=System.Data.SQLite.EF6;provider connection string=&quot;data source=C:\Users\Ron\Desktop\SqliteDemo\SqliteDemo\Northwind.sqlite;version=3&quot;" providerName="System.Data.EntityClient" />
</connectionStrings>
```

`<provider invariantName="System.Data.SQLite.EF6">` 中 `invariantName` 會被以下幾個地方參考, 要換名字的話四個地方都要換, 不然執行階段會拋例外 
+  `<DbProviderFactories>` 下的 `<add>` 的屬性 `invariant`  
+  `<connectionStrings>` 下的 `<add>` 的屬性 `connectionString` 內的 `provider` 
+  Northwind.edmx 中 `<Schema>` 的屬性 `Provider`

### 結論
這種做法對於快速開發應用來說很適合, 畢竟工具已經幫忙產出大部分的設定與程式碼.  

但也由於很多內容是自動產生的, 所以如果沒有特別去研究, 會不知道究竟自動產生了什麼設定與程式, 不知道相關細節修改起來就容易出錯, 而且自動產生 *.edmx 還要看所用版本的 visual stuio 有沒有支援, 如果用到不支援的版本就會變得很麻煩.  

### 參考
[EF6 workflow with SQLite DDEX provider](https://github.com/ErikEJ/SqlCeToolbox/wiki/EF6-workflow-with-SQLite-DDEX-provider#install-systemdatasqlite-nuget-package)  
[SQLite](https://system.data.sqlite.org/index.html/doc/trunk/www/downloads.wiki)  
[SQLite EntityFramework 6 Tutorial](https://erazerbrecht.wordpress.com/2015/06/11/sqlite-entityframework-6-tutorial/)  
[northwind-SQLite3](https://github.com/jpwhite3/northwind-SQLite3)  