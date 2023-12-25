---
title: 在 EF Core 中自定義 ValueConverter
date: 2023-12-25 21:49:57
categories:
- Tools
tags:
---

關於 EF Core 中的值轉換 (Value Conversion)，官方文件已經展示的非常詳細，但這樣的實作使得值轉換和 `DbContext` 高度相依，在中小型專案中這樣已經足夠，但在複雜的產品群中，如果需要基於 EF Core 建立內部共用的套件，就會難以將這些客製化的 `ValueConverter` 從套件中抽離，也難以讓應用程式自行擴充自己特殊的 `ValueConverter`，本文主要提供另外一個面向的實作來達到更低的耦合度與更高的可擴充性。

<!--more-->

### 前言
通常我們不需要自己客製化 Value Converters，但實務環境總是不會這麼單純。  

以一個實務的例子來說，舊的資料庫在將真假值欄位的值存成 1 代表 false，2 代表 true，然後在程式中用一個列舉 `TrueFalseEnum` 來表達時，而同一個資料庫又有正常的真假值欄位是存成 0 代表 false 而 1 代表 true，並在程式中轉成 `bool`。 當有機會重整舊系統時，想要統一這種真假值的型別但又因為要避免未重整部分出錯而不能直接修正資料庫的資料時，就需要在 EF Core 轉換過程直接透過不同的 `ValueConverter` 來讓這兩種情境都能和 `bool` 互相轉換。  

或是另外一種資料庫設計不當的情境，例如一張表將時間區間欄位設計成數字代表小時，另外一張表的時間區間欄位可能是代表分，這種不一致很容易造成程式中操作的困難，維護的工程師常常需要分心想現在這個數字代表的是時、分還是秒，也容易不小心弄錯產生 bug。 這時候不管是新專案還是重整舊系統，都會面臨到資料庫欄位變更困難但又不想讓程式遷就資料庫不當設計的兩難。 這時候也很適合用不同的 Value Converters 來轉換數字和 `TimeSpan` 之類的一致的型別，提升程式碼的可維護性。  

抽象來說，就是當資料庫和程式中的型別無法用預設機制轉換時，就可以考慮客製化 Value Converters.

### 實作
#### 另外繼承並實作 `ValueConverter<>`
首先我們可以參考 EF Core 內建的各種 Value Converters 來實作，例如：[DateTimeToTicksConverter.cs](https://github.com/dotnet/efcore/blob/main/src/EFCore/Storage/ValueConversion/DateTimeToTicksConverter.cs)。  

#### 提供一個客製的 Attribute
提供一個 `ValueConverterAttribute`，讓使用者可以放在 Entity Domain Model 中如下：
``` csharp
[Table("PERSON")]
public class Person
{
    [Column("WHATEVER")]
	public bool Whatever { get; set; }

    [Column("IS_DELETED")]
	[ValueConverter(typeof(BooleanToTrueFalse))]
    public bool IsDeleted { get; set; }
}
```

`Whatever` 欄位就是正常的布林值 (0 和 1)，而 `IsDeleted` 標記套用 `BooleanToTrueFalse` 這個 Value Converter 來適應資料庫中不當的真假值 (1 和 2)，在程式中我們可以一致的操作 `bool`，而資料庫的整理則可以等程式整理完後再另外分案處理。  

#### `DbContext` 中的實作
這是非常重要的一部份，如果我們照官方文件的作法，會讓 `DbContext` 和 `ValueConverter` 直接且高度依賴，這邊我們透過 `ValueConverterAttribute` 來解除他們的依賴，程式碼範例如下：

``` csharp
public class MyDbContext : DbContext
{
    private string _entityAssemblyName;

    public MyDbContext(
        DbContextOptions<MyDbContext> options,
        Setting settings)
        : base(options)
    {
        _entityAssemblyName = settings.EntityAssemblyName;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var entityTypes = Assembly.Load(_entityAssemblyName).GetTypes();
        foreach (var type in entityTypes)
        {
            var entity = modelBuilder.Entity(type);
            foreach (var property in type.GetProperties())
            {
                var converterAttribute = property.GetCustomAttribute<ValueConverterAttribute>();
                if (converterAttribute != null)
                {
                    var converter = (ValueConverter)Activator.CreateInstance(converterAttribute.Type);
                    entity.Property(property.Name).HasConversion(converter);
                }
            }
        }
    }
}
```

我們可以在 `OnModelCreating` 中透過偵測 `ValueConverterAttribute` 以及反射來設定 Value Conversions。 要注意這個只是用來展示概念，細節部分還需要視情況仔細設計。  

### 其他應用
這種設計不限於本文提到的 Value Converters 的情境，只要是會在 `MyDbContext` 中的各方法中和外部 (例如 Entity Domain Model) 產生直接依賴的情境，都可以用這種設計來解耦。  

### 注意事項
以下是一些重要的注意事項：  
1. 直接搜尋組件來找出標有特定 Attribute 的成員是效能很差的做法，在本例中可以接受是因為 `OnModelCreating` 只會在初始化時執行一次，整體效能影響不大，加上我遇到的是非常重視解耦與彈性的情境，所以是利大於弊的。
2. 這種設計不適用於無跨專案共用需求的 `DbContext`。

### 結論
基礎用法其實官方文件都有，但是實務上往往比文件上的範例複雜，這時候就需要變通並試著找出其他更適合的解決方案，但也要了解到不同的機制各有不同的缺點，需要謹慎評估。

### 參考
[Value Conversions](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions?tabs=data-annotations)  

[ValueConversion on GitHub](https://github.com/dotnet/efcore/tree/main/src/EFCore/Storage/ValueConversion)