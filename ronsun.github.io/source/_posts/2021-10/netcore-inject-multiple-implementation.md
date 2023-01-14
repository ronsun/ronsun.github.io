---
title: .NET Core 中單一介面多實作搭配依賴注入的幾種方式
date: 2021-10-31 21:16:36
categories:
- C#
- .NET Core
tags:
---

在 .NET Core 中注入多個實作有幾種方式，各自有不同的優缺點與試用情境，相關範例程式碼會放在 [這個 GitHub 專案](https://github.com/ronsun/Demo/tree/master/MultipleImplementation) 上。  

<!--more-->

### Implementation Factory
Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions 提供的多載，以 AddScoped 為例：   
`public static IServiceCollection AddScoped<TService>(this IServiceCollection services, Func<IServiceProvider, TService> implementationFactory) where TService : class`  

第二個參數提供一個工廠方法來決定要使用哪種實作。  

**優點**  
+ 非常簡單。

**缺點與限制**
+ 只適用於在注入時就知道要用什麼規則決定使用哪個實作。
+ 彈性低，雖然簡單但是適用情境狹窄。

### 注入 IEnumerable<T>
註冊多個實作，使用時注入 `IEnumerable<I>` 後再決定要使用哪個實作，可以搭配一個欄位用來標示該實作的名字 (要用類別名稱來標示也可以)。  

註冊服務：  
``` csharp
public void ConfigureServices(IServiceCollection services)
{
	services.AddControllers();

	services.AddSingleton<IFoo, Foo1>();
	services.AddSingleton<IFoo, Foo2>();
}
```

注入與使用：  
``` csharp
[ApiController]
[Route("[controller]")]
public class DemoController : ControllerBase
{
	private IEnumerable<IFoo> _allFoo;

	public DemoController(IEnumerable<IFoo> allFoo)
	{
		_allFoo = allFoo;
	}

	[HttpGet]
	public ActionResult<string> Get(string name)
	{
		var sb = new StringBuilder();

		var foo = _allFoo.FirstOrDefault(r => r.Name == name);
		sb.Append(foo?.Hi());

		return Ok(sb.ToString());
	}
}
```

**優點**  
+ 簡單。
+ 彈性高。

**缺點與限制**
+ 因為是注入所有的實作後再挑出要使用的那一個，其他的都用不到，會有浪費資源的疑慮，所以通常搭配 AddSingleton 使用。
+ 每次搜尋 ( `_allFoo.FirstOrDefault(r => r.Name == name)` )的時間複雜度都是 O(n)，需要多次搜尋時這個缺點會更明顯。

### Dictionary
註冊成一個 Dictionary 或類似的物件。  

註冊服務：  
``` csharp
public void ConfigureServices(IServiceCollection services)
{
	services.AddControllers();

	services.AddScoped<IBar, Bar1>();
	services.AddScoped<IBar, Bar2>();
	services.AddScoped<IReadOnlyDictionary<string, IBar>>(provider =>
	{
		var allBar = provider.GetService<IEnumerable<IBar>>();
		return allBar.ToDictionary(bar => bar.Name, bar => bar);
	});
}
```

注入與使用：  
``` csharp
[ApiController]
[Route("[controller]")]
public class DemoController : ControllerBase
{
	private IReadOnlyDictionary<string, IBar> _allBar;

	public DemoController(IReadOnlyDictionary<string, IBar> allBar)
	{
		_allBar = allBar;
	}

	[HttpGet]
	public ActionResult<string> Get(string name)
	{
		var sb = new StringBuilder();

		// bar
		_allBar.TryGetValue(name, out IBar bar);
		sb.Append(bar?.Hello());

		return Ok(sb.ToString());
	}
}
```

**優點**  
+ 簡單。
+ 彈性高。
+ 可擴充性高，如果是作為一個套件，即使沒提供適合的實作，使用者仍然可以自己實作 `IBar` 後注入。
+ 只有第一次初始化的時候將 `IEnumerable<T>` 轉成 Dictionary 時比較耗時，之後每次取用的時間複雜度都是 O(1)，在生命週期範圍內多次取用的效能優於 `IEnumerable<T>`。

**缺點與限制**
+ 沒有那麼直覺，對於使用者來說要記得注入時要注入為 `IReadOnlyDictionary<string, IBar>`。


### 透過另一個類別來轉接 (橋接模式)
這種做法的概念在於提供一個橋接用的類別來使用那些實作。  


註冊服務：  
``` csharp
public void ConfigureServices(IServiceCollection services)
{
	services.AddControllers();

	services.AddScoped<IBaz, Baz1>();
	services.AddScoped<IBaz, Baz2>();
	services.AddScoped<IBazBridge, BazBridge>();
}
```

注入與使用：  
``` csharp
[ApiController]
[Route("[controller]")]
public class DemoController : ControllerBase
{
	private IBazBridge _bazBridge;

	public DemoController(IBazBridge bazBridge)
	{
		_bazBridge = bazBridge;
	}

	[HttpGet]
	public ActionResult<string> Get(string name)
	{
		var sb = new StringBuilder();

		// baz
		sb.Append(_bazBridge.Hey(name));

		return Ok(sb.ToString());
	}
}
```

Bridge 相關類別：  
``` csharp

public interface IBazBridge
{
    string Hey(string name);
}

internal class BazBridge : IBazBridge
{
    private IReadOnlyDictionary<string, IBaz> _allBazDictionary;

    public BazBridge(IEnumerable<IBaz> allBaz)
    {
        _allBazDictionary = allBaz.ToDictionary(baz => baz.Name, baz => baz);
    }

    public string Hey(string name)
    {
        _allBazDictionary.TryGetValue(name, out IBaz baz);
        return baz?.Hey();
    }
}
```

> 如果有多個情境要使用 Bridge，可以將建構子中轉換 `IEnumerable<T>` 為 Dictionary 的部分抽成 `BridgeBase`，提供給所有 Bridge 繼承。

**優點**  
+ 彈性極高，Brige 中甚至可以做各種轉接與變化。  
+ 可擴充性高，如果是作為一個套件，即使沒提供適合的實作，使用者仍然可以自己實作 `IBaz` 後注入。
+ 只有第一次初始化的時候將 `IEnumerable<T>` 轉成 Dictionary 時比較耗時，之後每次取用的時間複雜度都是 O(1)，在生命週期範圍內多次取用的效能優於 `IEnumerable<T>`。
+ 相較於之前的注入 Dictionary 的解法，這種高彈性/擴充性的作法更適合放在套件中，這種設計相似於 `HttpClient` 與 `IHttpClientFactory` 之間的關係，所以相對於之前 Dictionary 的解法來說，使用上不直覺的問題相對低了一些。  

**缺點與限制**
+ 複雜很多。
+ 多個介面需要套用時會需要很多組 Bridge。
+ 註冊較不友善，`services.AddScoped<IBazBridge, BazBridge>();` 這邊使用者要認識具體實作 `BazBridge`。


### 透過另一個類別來管理 (Provider)
這種做法的概念在於提供一個管理用的類別來提供那些實作。  


註冊服務：  
``` csharp
public void ConfigureServices(IServiceCollection services)
{
	services.AddControllers();

	services.AddScoped<IQux, Qux1>();
	services.AddScoped<IQux, Qux2>();
	services.AddScoped<IProvider<IQux>, Provider<IQux>>();
}
, ``

注入與使用：  
``` csharp
[ApiController]
[Route("[controller]")]
public class DemoController : ControllerBase
{
	private IProvider<IQux> _quxProvider;

	public DemoController(IProvider<IQux> quxProvider)
	{
		_quxProvider = quxProvider;
	}

	[HttpGet]
	public ActionResult<string> Get(string name)
	{
		var sb = new StringBuilder();

		// baz
		sb.Append(_quxProvider.Get(name).Yo());

		return Ok(sb.ToString());
	}
}
```

Provider 相關類別：  
``` csharp

public interface IProvider<T>
    where T : INameable
{
    T Get(string name);
}

public class Provider<T> : IProvider<T>
    where T : INameable
{
    private IReadOnlyDictionary<string, T> _dictionary;

    public Provider(IEnumerable<T> nameable)
    {
        _dictionary = nameable.ToDictionary(n => n.Name, n => n);
    }

    public T Get(string name)
    {
        _dictionary.TryGetValue(name, out T imp);
        return imp;
    }
}

```

**優點**  
+ 有幾乎等同 `Bridge` 解法的優點 (除了彈性略低)。 
+ 泛用性極高，可重複套用在任何單介面多實作的情境。

**缺點與限制**
+ 因為不需要實作各種 `Bridge`，彈性較 `Bridge` 解法低一點。  
+ 註冊較不友善，`services.AddScoped<IProvider<IQux>, Provider<IQux>>();` 這邊使用者要認識具體實作 `Provider<T>`，且兩層泛型參數比較不好閱讀。  

### 綜合以上最佳化使用者體驗
這是我用在一個套件 - [MoreNet.DependencyInjection](https://github.com/ronsun/MoreNet.DependencyInjection) 上的作法，介紹直接看 GitHub。 

**優點**  
+ 泛用性極高，可重複套用在任何單介面多實作的情境。
+ 註冊簡易，對使用者比較友善。

**缺點與限制**
+ 實作很複雜。



### 結論
方法很多種，大致可以分為兩個面向
1. 注入全部後再挑選。
2. 提供管理用的類別 (不論是工廠/橋接或是其他模式)。

上面的範例只是為了展示這兩個大方向，實際運用時需要視情境變化調整，例如[這篇文章提到幾種做法](https://medium.com/geekculture/net6-dependency-injection-one-interface-multiple-implementations-983d490e5014)，都和上面範例不太一樣但大方向是一致的。  

### 參考
[Service registration methods](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-5.0#service-registration-methods-1)  

[.NET6 Dependency Injection — One Interface, Multiple Implementations](https://medium.com/geekculture/net6-dependency-injection-one-interface-multiple-implementations-983d490e5014)
