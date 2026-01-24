---
title: 建構子注入的潛在效能風險
date: 2026-01-25 01:15:46
categories:
- C#
- .NET Core
tags:
---

在 .NET Core 做 DI 時，大多習慣用建構子注入（Constructor Injection）。但如果依賴的 implementation factory 裡面藏著高成本的初始化，就可能在注入服務時初始化非必要的高成本依賴，進而造成效能問題。這篇整理問題成因與幾個可行的解法。

<!--more-->

### 問題描述
假設有個 `MyService` 依賴資料庫套件 `IDbClient`，而 `IDbClient` 的初始化非常耗能（例如建立連線）。而 `MyService` 有兩個方法，一個是依賴 IDbClient 但不常觸發的方法，另一個是無依賴但頻繁觸發的方法：  
``` csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<IDbClient>(sp =>
        {
            var client = new ActualDbClient("connection string");
            // High-cost operation
            client.OpenConnection();
            return client;
        });

        services.AddScoped<MyService>();
    }
}

public class MyService
{
    private readonly IDbClient _dbClient;

    public MyService(IDbClient dbClient)
    {
        _dbClient = dbClient;
    }

    // Low frequency; depends on IDbClient
    public string GetDataFromDb() => _dbClient.Query("SELECT * FROM Table");

    // High frequency; no dependency on IDbClient
    public int Calculate(int a, int b) => a + b;
}
```

當 `MyService` 是 Transient 或 Scoped，每次解析 `MyService` 都會觸發 `IDbClient` 初始化。若平常多半只會叫到低成本的 `Calculate()`，卻也被迫初始化高成本的 `IDbClient`，就會白白浪費效能。  

### 解決方案

#### 方案一：注入`IServiceProvider` 隨需取用（依賴不透明）
``` csharp
public class MyService
{
    private readonly IServiceProvider _serviceProvider;

    public MyService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public string GetDataFromDb()
    {
        var dbClient = _serviceProvider.GetRequiredService<IDbClient>();
        return dbClient.Query("SELECT * FROM Table");
    }

    public int Calculate(int a, int b) => a + b;
}
```

這做法類似 Service Locator Pattern，雖然這常被視為 Anti-Pattern，但 .NET 把它封裝成 `IServiceProvider` 介面，顧慮少了不少。但由於不在建構子明列依賴，會導致專案風格不一致，以及依賴不透明。

#### 方案二：根據依賴拆分 Service（分類混亂風險）
``` csharp
public class MyCalculationService
{
    public int Calculate(int a, int b) => a + b;
}

public class MyDbService
{
    private readonly IDbClient _dbClient;

    public MyDbService(IDbClient dbClient)
    {
        _dbClient = dbClient;
    }

    public string GetDataFromDb() => _dbClient.Query("SELECT * FROM Table");
}
```

這是把原本的 `MyService` 根據依賴拆成兩支獨立 Service。雖然可行但久了可能分類標準愈分愈亂（有時看功能、有時看依賴），對長期維護不好。

#### 方案三：輕量化 Implementation Factory (最推薦）
核心思路是讓 implementation factory 保持輕量，把耗能的初始化延後到真的要用時才做。

``` csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<IDbClient>(sp =>
        {
            var client = new ActualDbClient("connection string");
            // Avoid expensive initialization here
            // client.OpenConnection();
            return client;
        });

        services.AddScoped<MyService>();
    }
}
```

這個方案從根本上解決問題，讓 Implementation Factory 的執行保持輕量，真正耗能的初始化延遲到實際需要時才進行。這樣即使 `MyService` 被頻繁建立，也不會觸發高成本的初始化操作。

### 結論
建構子注入是 .NET Core DI 的慣例做法，但當依賴的初始化成本較高時，需要特別注意是否會造成不必要的效能浪費。三種解決方案各有優缺：

| 方案 | 優點 | 缺點 |
|------|------|------|
| 注入 `IServiceProvider` | 實作簡單 | 依賴不透明，風格不一致 |
| 拆分 Service | 職責明確 | 分類標準混亂，維護困難 |
| 輕量 Implementation Factory | 從根本解決，維持慣例 | 如果套件不支援，就要換套件或另外設計來避開問題 |

建議優先採用方案三，從根本上確保 Implementation Factory 是輕量的操作。若真的無法達成，再考慮方案一或方案二作為特殊場景下的替代方案。

### 參考
實務經驗。