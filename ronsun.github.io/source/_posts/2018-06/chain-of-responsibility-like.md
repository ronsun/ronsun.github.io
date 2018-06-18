---
title: 責任鏈模式(Chain of Responsibility)的變形
date: 2018-06-18 15:20:30
categories:
- DesignPatterns
tags:
---

責任鏈模式在教科書上的典型範例是用在簽核流程上, 也就是一個任務/資料會依序流過各個處理節點, 每個節點會判斷任務/資料是不是在自己的權責範圍內, 決定要自己處理還是繼續往下拋.  

前陣子公司專案剛好有一個類似情境, 但需求與教科書用法不同, 第一時間想到一個變形的作法, 而公司大神也提供了另一個更簡潔的做法, 趁還沒忘整理一下記下來.

> [完整的示範專案和文件](https://github.com/ronsun/Demo/tree/master/ValidatorManager)

<!--more-->

情境是這樣的, 有一系列的交易驗證機制, 在很多地方都會被用到, 規則是先檢查A, 若A通過再檢查B, 若B通過再檢查C...依此類推, 直覺呼叫各個檢查點的話, 程式碼會顯得有點複雜.

### 從層層上報變成生產線
改造一下責任鏈, 把行為變成讓一個任務/資料會依序流過各個處理節點, **各節點處理自己該做的部分, 沒問題再往下拋**, 如下面範例:

**待驗證內容**
``` csharp
public class ValidateContext
{
    public decimal Amount { get; set; }

    public string Email { get; set; }

    public string Country { get; set; }
}
```

**各個驗證節點** : 每個驗證點都繼承自 `Validator`.
``` csharp
public abstract class Validator
{
    public Validator NextValidator { get; set; }

    public abstract bool Validate(ValidateContext context);
}

public class AmountValidator : Validator
{
	public override bool Validate(ValidateContext context)
	{
        if (context.Amount > 10 && context.Amount < 1000)
        {
            return NextValidator?.Validate(context) ?? true;
        }

        return false;
	}
}

public class EmailValidator : Validator
{
    public override bool Validate(ValidateContext context)
    {
        if (context.Email == "ron.sun@mailserver.com")
        {
            return NextValidator?.Validate(context) ?? true;
        }

	    return false; 
    }
}

public class CountryValidator : Validator
{
    public override bool Validate(ValidateContext context)
    {
        if (context.Country == "Taiwan")
        {
            return NextValidator?.Validate(context) ?? true;
        }

	    return false; 
    }
}
```

**驗證流程管理者** : 這邊有些教學會把他直接放在呼叫端, 但如果要重用的話, 還是拉出來比較適合. 
``` csharp
public class ValidatorManager
{
    public bool BasicValidation(ValidateContext context)
    {
        var amountValidator = new AmountValidator();
        var emailValidator = new EmailValidator();

        Validator rootValidator = amountValidator;
        amountValidator.NextValidator = emailValidator;

        return rootValidator.Validate(context);
    }

    public bool FullValidation(ValidateContext context)
    {
        var amountValidator = new AmountValidator();
        var emailValidator = new EmailValidator();
        var countryValidator = new CountryValidator();

        Validator rootValidator = amountValidator;
        amountValidator.NextValidator = emailValidator;
        emailValidator.NextValidator = countryValidator;

        return rootValidator.Validate(context);
    }
}
```

**呼叫端**
``` csharp
static void Main(string[] args)
{
    var mng = new ValidatorManager();
    var context = new ValidateContext() { Amount = 100};
    bool isValid = mng.Validate(context);
    if (isValid)
    {
        Console.WriteLine("pass validations.");
    }

    Console.ReadLine();
}
```

> 視情境還是可以再變化, 例如: 驗證不通過時不中斷, 一定要跑完所有驗證才返回結果與錯誤訊息總結, 那就是把返回從 `bool` 改成一個物件, 讓每個驗證節點去操作返回物件. 

### List 和 Func 搭配迴圈
委派, 集合與迴圈的搭配, 可以讓一連串的方法呼叫變得更簡潔, 也有一點責任鏈的味道在裡面 , 範例如下:

**驗證方法**
``` csharp
public class Validator
{
    public bool AmountValidate(decimal amount)
    {
        return amount > 10 && amount < 1000;
    }

    public bool EmailValidate(string email)
    {
        return email == "ron.sun@mailserver.com";
    }

    public bool CountryValidate(string country)
    {
        return country == "Taiwan";
    }
}
```

**呼叫端**
``` csharp
static void Main(string[] args)
{
    var context = new ValidateContext() { Amount = 100 };
    var validator = new Validator();
    var validateList = new List<Func<bool>>()
    {
        () => validator.AmountValidate(100),
        () => validator.EmailValidate("name@mail.com"),
        () => validator.CountryValidate("Taiwan")
    };

    foreach(var item in validateList)
    {
        var isValid = item();
        if (!isValid) break;
    }

    Console.ReadLine();
}
```

這種做法讓集合內的 `Func<>`, `Action<>` 等委派方法能依序被處理, 且這些方法可以分別在不同的類別裡, 也不用像一般責任鏈必須衍生自父類別, 唯一的限制就是 `Func<>`, `Action<>` 的型別參數必須完全相同. 

### 結論
責任鏈的基本樣貌就像一條生產線, 基於這個原則下衍生的變形或簡化做法其實還不少, 我把他們稍微整理在[這個專案裡](https://github.com/ronsun/ValidatorManager).
