---
title: 使用 Aggregate Services 收斂依賴注入的物件
date: 2020-04-03 00:38:03
categories:
- C#
tags:
---

這篇是為了紀錄依賴注入時, 利用建構子注入的情境下, 當注入到有繼承關係的類別時所引發的維護上的困擾與解決方案.  

<!--more-->

### 問題背景
考慮下面的程式碼  

``` csharp
public class BaseServcie
{
    private IFooDAO _fooDAO;

    public BaseServcie(IFooDAO fooDAO)
    {
        _fooDAO = fooDAO;
    }
}

public class DerivedService : BaseServcie
{
    private IBarDAO _barDAO;

    public DerivedService(IFooDAO fooDAO, IBarDAO barDAO)
        : base(fooDAO)
    {
        _barDAO = barDAO;
    }
}
```

由於這個情境下, 父類別 (基底類別) 只有一個有參數的建構子, 所以不會從編譯器取得公開無參數建構子 (註1) , 而子類別 (衍生類別) 的建構子必須明確指定所要使用的父類別的建構子.  

> 註1:  
> [Unless the class is static, classes without constructors are given a public parameterless constructor by the C# compiler in order to enable class instantiation](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/using-constructors)  

這樣的問題在於, 當父類別的建構子參數數量改變時, 所有子類別的建構子要同時修改, 如果子類別很多的話, 必然會造成維護上的困擾. 直覺上當然可以改用其他注入方式, 例如直接從 DI 套件提供的 service provider 去取得實體, 但這樣會有比較難做單元測試的缺點, 不是優先選項.  

### Aggregate Service
要避免這樣的問題的切入點就是 `讓父類別的參數數量固定一個就好`, 也就是 Aggregate Service 的概念, 這樣子不管之後要怎麼改動父類別所依賴的物件數量, 不論是父類別還是廣大的子類別們都不需要修改任何建構子的參數列.  

#### 通用作法
如下程式碼: 

``` csharp
public class BaseServcie
{
    private IFooDAO _fooDAO;
    public BaseServcie(AggregateBaseServiceDAO aggregateDAO)
    {
        _fooDAO = aggregateDAO.FooDAO;
    }
}

public class DerivedService : BaseServcie
{
    private IBarDAO _barDAO;
    public DerivedService(AggregateBaseServiceDAO aggregateDAO, IBarDAO barDAO)
        : base(aggregateDAO)
    {
        _barDAO = barDAO;
    }
}

public class AggregateBaseServiceDAO
{
    public AggregateBaseServiceDAO(IFooDAO _fooDAO)
    {
        FooDAO = _fooDAO;
    }

    public IFooDAO FooDAO { get; }
}
```

`IFooDAO` 收納進 `AggregateBaseServiceDAO` 成為一個屬性, 就算之後 `BaseServcie` 需要增加其他的依賴, `BaseServcie` 和 `DerivedService` 的建構子也都不會受到影響.  
這個通用做法也不是什麼特別的概念, 就是在建構子注入的基礎下加上一點變化而已, 理論上也不會受限於使用哪一個 DI 套件.  

> 如果沒必要的話, 不一定要多新增一個介面 `IAggregateBaseServiceDAO` 來讓 `AggregateBaseServiceDAO` 實作.  

#### Autofac 的 RegisterAggregateService
Autofac 提供了比較方便的用法, Aggregate Service 簡化成如下面這樣, 但是 **Aggregate Service 必須是介面**, 且除了 Autofac 本身外需要安裝另外一個套件 Autofac.Extras.AggregateService.  

``` csharp
public interface IAggregateBaseServiceDAO
{
    IFooDAO FooDAO { get; }
}
```

另外, 註冊的時候要改用 `RegisterAggregateService` 方法如下:  
``` csharp
builder.RegisterAggregateService<IAggregateBaseServiceDAO>();
```

這邊可以看到 Autofac 簡化了 Aggregate Service 內部的建構子注入的部分, 讓使用的時候不會覺得為了封裝這些物件而需要多做很多事, 限制 Aggregate Service 必須是介面且不需要實作類別, 某種程度上也有暗示後面維護者不要去誤用它的效果, 比起前面介紹的通用型 Aggregate Service 來說優點更多.  

### 結論
總體來說, 如果用 Autofac 且多安裝一個套件 Autofac.Extras.AggregateService 不會造成困擾的話, 用 Autofac 的作法會比較漂亮一點. 但如果 DI 套件沒有相關功能的話, 用通用的作法也是很不錯的.  

### 參考
[Autofac Aggregate Services](https://autofaccn.readthedocs.io/en/latest/advanced/aggregate-services.html)  