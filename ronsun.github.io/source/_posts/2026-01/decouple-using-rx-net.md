---
title: 使用 Rx.NET 解耦服務之間的依賴
date: 2026-01-01 01:14:15
categories:
- C#
- Packages
tags:
---

在典型的分層式架構下，遇上複雜的業務邏輯時，服務層的不同服務間仍然很容易產生依賴。這篇文章會循序漸進地說明依賴關係的問題，以及如何使用 Rx.NET（Reactive Extensions for .NET）優雅地解耦這些依賴。  

<!--more-->

### 問題：高度耦合

[高度依賴版本](https://github.com/ronsun/BlogDemo.DecoupleRxNET/tree/v1)  

在沒有使用任何模式的情況下，訂單服務（`OrderService`）需要直接依賴 `IStockService` 和 `IAccountingService`：  

``` csharp
public class OrderService(
    ILogger<OrderService> logger,
    IStockService stockService,
    IAccountingService accountingService)
    : IOrderService
{
    public void Confirm(Guid orderId)
    {
        logger.LogInformation($"Order confirming: {orderId}");

        stockService.Ship(Guid.NewGuid(), 10);

        accountingService.Credit(100m, PaymentMethod.CreditCard);

        logger.LogInformation($"Order confirming: {orderId}, doing something else.");

        logger.LogInformation($"Order confirmed: {orderId}");
    }
}
```

這種設計方式存在幾個問題：  
1. **強依賴**：`OrderService` 直接依賴了 `IStockService` 和 `IAccountingService`。在分層架構下同層間水平依賴是一個不好的現象。
2. **循環依賴風險**：在大型專案中，依賴管理不當容易產生循環依賴。

> 在 Controller 中組合不同的服務是一種可行方案，但不總是適當。例如，在執行 Confirm 方法的中途需要呼叫其他服務，之後繼續執行剩餘邏輯，此時把 Confirm 方法拆分成多個方法就不適合了。

### 方案一：使用 Rx.NET 發佈訂閱（含副作用）

[有副作用的解耦版本](https://github.com/ronsun/BlogDemo.DecoupleRxNET/tree/v2)  

在這個版本中，我們引入 Rx.NET 的 `Subject<T>`。由訂閱者（Controller 中的訂閱相關程式碼）來管理訂閱細節，從 `OrderService` 發佈訂單確認的事件，這樣 `OrderService` 就可避免直接依賴其他服務。  

**修改後的 Controller**  

``` csharp
[ApiController]
[Route("[controller]/[Action]")]
public class OrderController(
    IOrderService orderService,
    IStockService stockService,
    IAccountingService accountingService)
    : ControllerBase
{
    [HttpGet]
    public IActionResult Confirm()
    {
        var subject = new Subject<OrderConfirming>();

        // Stock
        subject.Subscribe(topic =>
        {
            foreach (var product in topic.Products)
            {
                stockService.Ship(product.Id, product.Quantity);
            }
        });

        // Accounting
        subject.Subscribe(topic =>
        {
            foreach (var product in topic.Products)
            {
                var amount = product.UnitPrice * product.Quantity;
                accountingService.Credit(amount, product.PaymentMethod);
            }
        });

        orderService.Confirm(Guid.NewGuid(), subject);
        return Ok();
    }
}
```

**修改後的 OrderService**  

``` csharp
public class OrderService(
    ILogger<OrderService> logger)
    : IOrderService
{
    public void Confirm(Guid orderId, IObserver<OrderConfirming> observer)
    {
        logger.LogInformation($"Order confirming: {orderId}");

        var message = new OrderConfirming
        {
            Products = new List<OrderConfirming.Product>
            {
                new OrderConfirming.Product
                {
                    Id = Guid.NewGuid(),
                    Quantity = 10,
                    PaymentMethod = PaymentMethod.CreditCard,
                    UnitPrice = 9.99m,
                },
                new OrderConfirming.Product
                {
                    Id = Guid.NewGuid(),
                    Quantity = 5,
                    PaymentMethod = PaymentMethod.CashOnDelivery,
                    UnitPrice = 19.99m,
                }
            }
        };

        observer.OnNext(message);

        logger.LogInformation($"Order confirming: {orderId}, doing something else.");

        logger.LogInformation($"Order confirmed: {orderId}");
    }
}
```

**優點**：  
- `OrderService` 不再依賴具體的服務，只依賴 `IObserver<T>`  
- 解耦了服務間的直接呼叫關係  

**缺點**：
1. **職責分離不清**：發佈訂閱邏輯散落在 Controller 中，無法完全職責分離
2. **維護性差**：相關程式碼分散在多個 Controller 中，修改時難以統一管理
3. **API 設計不夠簡潔**：`OrderService` 需要增加一個 `observer` 參數

### 方案二：優雅的解耦方案（推薦）

[優雅解耦版本](https://github.com/ronsun/BlogDemo.DecoupleRxNET/tree/v3)  

這個版本的核心概念是：**將訂閱邏輯獨立封裝成專門的訂閱者類別，透過 DI（依賴注入）將訂閱者注入到服務中**。  

**核心類別：ObserverBase<T>**  

首先，我們建立一個通用的觀察者基底類別：  

``` csharp
public class ObserverBase<T> : IObserver<T>
{
    private IObserver<T> _observerSubject;

    protected IObservable<T> ObservableSubject { get; }

    protected ObserverBase()
    {
        var subject = new Subject<T>();
        ObservableSubject = subject;
        _observerSubject = subject;
    }

    public void OnCompleted() => _observerSubject.OnCompleted();

    public void OnError(Exception error) => _observerSubject.OnError(error);

    public void OnNext(T value) => _observerSubject.OnNext(value);
}
```

**專門的訂閱者類別 OrderConfirmingSubscriber**  

``` csharp
public class OrderConfirmingSubscriber : ObserverBase<OrderConfirming>, IObserver<OrderConfirming>
{
    public OrderConfirmingSubscriber(
        IStockService stockService,
        IAccountingService accountingService)
    {
        // Stock
        base.ObservableSubject.Subscribe(topic =>
        {
            foreach (var product in topic.Products)
            {
                stockService.Ship(product.Id, product.Quantity);
            }
        });

        // Accounting
        base.ObservableSubject.Subscribe(topic =>
        {
            foreach (var product in topic.Products)
            {
                var amount = product.UnitPrice * product.Quantity;
                accountingService.Credit(amount, product.PaymentMethod);
            }
        });
    }
}
```

**簡化的 OrderService**  

``` csharp
public class OrderService(
    ILogger<OrderService> logger,
    IObserver<OrderConfirming> orderConfirmingObserver)
    : IOrderService
{
    public void Confirm(Guid orderId)
    {
        logger.LogInformation($"Order confirming: {orderId}");

        var message = new OrderConfirming
        {
            Products = new List<OrderConfirming.Product>
            {
                new OrderConfirming.Product
                {
                    Id = Guid.NewGuid(),
                    Quantity = 10,
                    PaymentMethod = PaymentMethod.CreditCard,
                    UnitPrice = 9.99m,
                },
                new OrderConfirming.Product
                {
                    Id = Guid.NewGuid(),
                    Quantity = 5,
                    PaymentMethod = PaymentMethod.CashOnDelivery,
                    UnitPrice = 19.99m,
                }
            }
        };

        orderConfirmingObserver.OnNext(message);

        logger.LogInformation($"Order confirming: {orderId}, doing something else.");

        logger.LogInformation($"Order confirmed: {orderId}");
    }
}
```

**簡化的 Controller**  

``` csharp
[ApiController]
[Route("[controller]/[Action]")]
public class OrderController(IOrderService orderService)
    : ControllerBase
{
    [HttpGet]
    public IActionResult Confirm()
    {
        orderService.Confirm(Guid.NewGuid());
        return Ok();
    }
}
```

**DI 設定**

``` csharp
public static void Main(string[] args)
{
    var builder = WebApplication.CreateBuilder(args);

    builder.Services.AddControllers();
    builder.Services.AddScoped<IOrderService, OrderService>();
    builder.Services.AddScoped<IStockService, StockService>();
    builder.Services.AddScoped<IAccountingService, AccountingService>();

    // 關鍵：注入 OrderConfirmingSubscriber
    builder.Services.AddScoped<IObserver<OrderConfirming>, OrderConfirmingSubscriber>();

    var app = builder.Build();
    app.UseAuthorization();
    app.MapControllers();
    app.Run();
}
```

**優點**：
1. **職責清晰**：Controller、Service、Subscriber 各司其職
2. **易於維護**：所有訂閱邏輯集中在 `OrderConfirmingSubscriber` 中，修改時無需觸及 Controller 和 Service
3. **簡潔的 API**：`OrderService.Confirm()` 簡潔，沒有額外的 observer 參數
4. **易於擴展**：新增其他訂閱者只需維護訂閱者類別，無需修改現有程式碼
5. **可測試性**：各個組件可以獨立測試

### 結論

通過對比三個版本，我們可以看到服務解耦的逐步演進：

| 特性 | V1（高度耦合） | V2（有副作用解耦） | V3（優雅解耦） |
|------|--------|--------|--------|
| 服務間依賴 | 強耦合 | 低耦合 | 低耦合 |
| 訂閱邏輯位置 | N/A | Controller | Subscriber 類別 |
| 職責分離 | 差 | 一般 | 優秀 |
| API 簡潔度 | 好 | 差 | 優秀 |
| 維護性 | 差 | 一般 | 優秀 |
| 可擴展性 | 差 | 一般 | 優秀 |


在大型專案中，雖然 V3 版本初看似更複雜，但帶來的可維護性、易管理性和可擴展性的提升，會在長期維護中帶來顯著的收益。

### 參考
- [dotnet/reactive GitHub](https://github.com/dotnet/reactive)
- [Introduction to Rx.NET 2nd Edition eBook](https://endjincdn.blob.core.windows.net/assets/ebooks/introduction-to-rx-dotnet/introduction-to-rx-dotnet-2nd-edition.pdf)
- [BlogDemo.DecoupleRxNET](https://github.com/ronsun/BlogDemo.DecoupleRxNET)
