---
title: C# 的 explicit 與 implicit 關鍵字
date: 2017-09-24 22:53:40
categories:
- C#
- Language Spec

---

來紀錄一下 C# 的兩個關鍵字 , `explicit` 以及 `implicit`。

<!--more-->

### 定義與使用時機

#### implicit
隱含轉換, 不需要明確指定轉換類型, 例如 `int` 轉換成 `float` 時, 只需要直接指定變數, 如下:  

``` csharp
int i = 10;
float f = i;
```

隱含轉換使用於轉換不會造成資料誤差或例外的情境, 以本例來說, `int` 轉換成 `float` 值不會有誤差, 但反過來就會, 所以 `float` 轉換成 `int` 時需要明確轉換。

#### explicit
明確轉換, 必須明確指定轉換類型, 同樣的例子, 當 `float` 要轉換成 `int` 的時候, 就會變成:  

``` csharp
float f = 10.12f;
int i = (int)f;
```

明確轉換使用於轉換可能會造成資料誤差或例外時, 以本例來說, 10.12 轉成 `int` 後, 結果是會不同的, 所以應該用明確轉換比較恰當。

### 自訂物件的隱含與明確轉換

#### 範例情境

首先我們先來考慮一個情境, 現在有兩個類別, 椅子 (Chair) 以及板凳 (Bench) 。

> 椅子只能坐一個人, 板凳則沒有限制, 所以椅子轉換成板凳時可以是隱含轉換, 但反過來就應該用明確轉換。  

先來個椅子, 有個 `HeadCount` 屬性, 當指定給他的值大於 1 的時候, 代表人數超載了, 會拋出一個例外, 且有一個隱含轉換的方法 `public static implicit operator Bench(Chair chair)` 如下:

``` csharp
public class Chair
{
    private int _headCount;
    public int HeadCount
    {
        get
        {
            return _headCount;
        }
        set
        {
            if (value > 1)
            {
                throw new ArgumentException("椅子座位只能容納一個人");
            }
            _headCount = value;
        }
    }

    // Chair 轉 Bench 可以是隱含轉換
    // 這個方法也可以放在 Bench 中, 但不能兩邊都寫, 編譯不會過
    public static implicit operator Bench(Chair chair)
    {
        return new Bench()
        {
            HeadCount = chair.HeadCount
        };
    }
}
```

再來是板凳, 板凳沒有人數限制, 有一個明確轉換的方法 `public static explicit operator Chair(Bench bench)` 如下:

``` csharp
public class Bench
{
    public int HeadCount { get; set; }

    // Bench 轉 Chair 需要明確轉換
    // 這個方法也可以放在 Chair 中, 但不能兩邊都寫, 編譯不會過
    public static explicit operator Chair(Bench bench)
    {
        return new Chair()
        {
            HeadCount = bench.HeadCount
        };
    }
}
```

用戶程式:
``` csharp
static void Main(string[] args)
{
    Chair chair = new Chair()
    {
        HeadCount = 1
    };

    Bench bench = new Bench()
    {
        HeadCount = 1
    };

    //當 bench 的 HeadCount 值大於1時, 這邊會拋 Exception 出來
    Chair chairFromBench = (Chair)bench;
    Bench benchFromChair = chair;
}
```

#### 使用說明
接著說明一下轉換的部分, 實作轉換方法有幾個限制  
+ 存取修飾詞必須是 public
+ 必須是靜態方法(static)
+ 必須有 implicit 或 explicit 關鍵字
+ 必須有 operator 關鍵字

總括來說方法必須長這樣
```
public static implicit operator Destination(Source src)
public static explicit operator Destination(Source src)

public static implicit operator Source(Destination dest)
public static explicit operator Source(Destination dest)
```

### 結論
implicit / explicit operator 可以寫在來源或目標中, 但是沒規範又容易亂, 目前看起來寫在來源中比較好, 設計類別 `A` 的時候由類別 `A` 決定自己可以隱含或是明確轉換成何種類別, 相較於寫在目標類別中來說比較合理.  

### 參考
[MS](https://docs.microsoft.com/zh-tw/dotnet/csharp/language-reference/keywords/conversion-keywords)  