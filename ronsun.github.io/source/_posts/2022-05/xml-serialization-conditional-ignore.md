---
title: XML 序列化時有條件的忽略欄位
date: 2022-05-30 00:10:41
categories:
- C#
- .NET
tags:
---

之前遇到需要在 XML 序列化的過程中忽略值為空值或預設值的欄位, 查了一下才發現不太好找而且做法沒有很直覺.

<!--more-->

### 作法
#### `bool ShouldSerialize{PropertyName}`  

建立回傳型別為 `bool` 的方法, 方法名稱必須為 ShouldSerialize 加上屬性名.  

``` csharp
public class M
{
    public string S { get; set; }
    public int N { get; set; }
    
    // ShouldSerialize + S
    public bool ShouldSerializeS()
    {
        return !string.IsNullOrEmpty(S);
    }
}

```
#### 唯獨屬性 `{PropertytName}Specified`  
建立唯獨屬性, 名稱必須為屬性名加上 Specified, 有幾個細節:
+ 不一定要是唯獨屬性, 有 setter 也可以, 但是沒意義, 不要這樣做.  
+ Attribute `System.Xml.Serialization.XmlIgnore` 不是必要的 (因為唯獨), 但建議要加, 避免有人加了 setter 或其他因素而導致他被序列化, 閱讀時也多一個提示效果.  

``` csharp
public class M
{
    public string S { get; set; }
    public int N { get; set; }
        
    [System.Xml.Serialization.XmlIgnore]
    public bool NSpecified
    {
        get 
        {
            return N > 0;
        }
    }
}
```

#### 呼叫端與綜合分析
簡單的呼叫端範例.  
``` csharp
public class Program
{
    public static void Main()
    {
        var m = new M();
        m.S = "";
        XmlSerializer xs = new XmlSerializer(typeof(M));
        StringWriter sw = new StringWriter();
        xs.Serialize(sw, m);
        Console.WriteLine(sw.ToString());
    }
}
```

> 建議:  
> 1. 優先使用 `bool ShouldSerialize{PropertyName}` 方法, 主要原因是因為他名字比較好記也比較直覺, 另外一個做法比較麻煩.
> 2. 將屬性和條件拆開成個獨立的檔案搭配 partial class, 範例如下: 
>    ``` csharp
>    // M.cs
>    public partial class M
>    {
>        public string S { get; set; }
>        public int N { get; set; }
>    }
>  
>    // M.Ignore.cs
>    public partial class M
>    {
>        public bool ShouldSerializeS()
>        {
>            return !string.IsNullOrEmpty(S);
>        }
>    }
>    ```

### 結論
找到解法的時候其實很驚訝, 因為這種做法維護上其實會有點困擾, 想想如果屬性要改名, 結果還要有意識的刻意找出 `ShouldSerialize{PropertyName}` 或是 `{PropertytName}Specified` 來修改, 對於維護其實不太友善, 只是沒有找到其他做法也就先拿來用了.  

### 參考
[Xml serialization - Hide null values](https://stackoverflow.com/a/5818571)  

[How to exclude null properties when using XmlSerializer](https://stackoverflow.com/a/1533339)  
