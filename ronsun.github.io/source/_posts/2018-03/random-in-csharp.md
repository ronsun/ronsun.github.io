---
title: C#中的隨機數
date: 2018-03-04 00:53:19
categories:
- C#
- .NET
tags:
- Random
- RNGCryptoServiceProvider

---

前些日子, 收到一個小需求需要隨機產生一組帶大小寫字母和數字的亂數字串, 想說需求滿簡單的, 快速寫一下就寫完commit了, 然後過不久就爆掉了.  
來看看究竟寫了些什麼鬼東西~

<!--more-->

``` csharp
public class RandomUtil
{
    private string _charDic = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    public string RandomString(int length)
    {
        Random rdm = new Random();
        string result = string.Empty;
        for (int i = 0; i < length; i++)
        {
            int nextIndex = rdm.Next(_charDic.Length);
            result += _charDic[nextIndex];
        }
        return result;
    }
}
```

程式內容很單純, 就是用`System.Random`隨機生出指定長度的字串, 但問題就出在需要隨機產生兩組, 於是呼叫端就呼叫了兩次, 像是這樣:  

```csharp
var rdm = new RandomUtil();
Console.WriteLine(rdm.RandomString(10));
Console.WriteLine(rdm.RandomString(10));
```

然後產生的兩個結果字串一模一樣, Why?

### 同時建立多個Random

`Random`的產生方式是基於一個種子來產生的, 也就是`public Random(int Seed)`中的`Seed`, 也就是說如果種子一樣, 那兩個new出來的Random物件產生的隨機數是一模一樣的, 而另外一個不帶參數的建構子呢?  
從[referencesource.microsoft.com](https://referencesource.microsoft.com/#mscorlib/system/random.cs)上面可以查到原始碼是這樣的:

``` csharp
public Random() 
        : this(Environment.TickCount) 
{
}
```

是的,預設以`Environment.TickCount`做為種子, 而`Environment.TickCount`是衍生自系統計時器的一個值. 

> **因為`Random()`的亂數是基於系統計時器產生的, 所以如果在極短時間內 (Environment.TickCount相同) 建立多個`Random`實例,就會導致產生的亂數是一樣的.**


知道問題後, 腦中閃過兩個做法:  

#### 解一: Thread.Sleep()
在`new Random()`之前, 先延時一毫秒, 避免拿到重複的時間, 雖然直覺但我個人不喜歡.

#### 解二: 建立唯一的Random

``` csharp
public class RandomUtil
{
    private string _charDic = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    private Random _rdm = new Random();

    public string RandomString(int length)
    {
        string result = string.Empty;
        for (int i = 0; i < length; i++)
        {
            int nextIndex = _rdm.Next(_charDic.Length);
            result += _charDic[nextIndex];
        }
        return result;
    }
}
```

`RandomUtil`初始化的時候就建立一個唯一的`Random`物件, 避免短時間內重複建立, 呼叫端程式碼不變, 這次兩次產生的結果是一樣的了,問題在**大部分**的情境下解決了.  


### 多執行緒環境下的Random

考慮同時有兩條執行緒都建立了`Random`, 簡單比對一下結果: 

```csharp
string firstThreadResult = null;
var firstThread = new Thread(new ThreadStart(() =>
{
    firstThreadResult = new RandomUtil().RandomString(10);
}));

string secondThreadResult = null;
var secondThread = new Thread(new ThreadStart(() =>
{
    secondThreadResult = new RandomUtil().RandomString(10);
}));
            
firstThread.Start();
secondThread.Start();

firstThread.Join();
secondThread.Join();
            
var rdm = new RandomUtil();
Console.WriteLine(firstThreadResult);
Console.WriteLine(secondThreadResult);
```

> **實驗結果顯示, 兩個產生的字串一樣, 所以在多執行緒的情境下, 還是有機會產生重複的亂數組合.**

#### 解三: RNGCryptoServiceProvider

``` csharp
public class RandomUtil
{
    private string _charDic = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private RNGCryptoServiceProvider _rng = new RNGCryptoServiceProvider();

    public string RandomString(int length)
    {
        string resultArr = string.Empty;

        for (int i = 0; i < length; i++)
        {
            var nextBytes = new byte[4];
            _rng.GetBytes(nextBytes);

            var index = BitConverter.ToInt32(nextBytes, 0) % _charDic.Length;
            resultArr += _charDic[index];
        }

        return resultArr;
    }
}
```

`RNGCryptoServiceProvider`可以避免`Random`在多執行緒情境下的重複問題, 但缺點就是他不像`Random`提供那麼多方法, 所以需要自己實作`Next()`,`Next(max)`,`Next(min, max)`等方法.

> 後來我把相關方法整理重構過放在[我的 Github 上](https://github.com/ronsun/LazyGuy/blob/master/LazyGuy/Utils/RandomValueGenerator.cs)了, 實作細節有不少差異, 但概念是跟上面的範例一樣的. 

### 延伸 - 關於隨機數
[密碼學(隨機數筆記)](https://dotblogs.com.tw/stanley14/2016/09/11/153133)

### 參考
[Random numbers - C# in depth](http://csharpindepth.com/Articles/Chapter12/Random.aspx)  
