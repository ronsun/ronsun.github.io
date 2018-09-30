---
title: XmlWriterSettings.Encoding 設定無效
date: 2018-11-01 20:21:45
categories:
- C#
- .NET
tags:
---

最近在寫一個比較方便使用的工具方法來做 XML 的序列化的時候, 發生了一個神奇的問題, `XmlWriterSettings.Encoding` 預設應該是 UTF-8, 但是轉出來的 XML 卻是 UTF-16, 這就奇了怪了...

<!--more-->

### XmlWriterSettings.Encoding 設定無效
先來看下面這段 Demo
``` csharp
private static string DemoForFailure()
{
    var xmlWriterSettings = new XmlWriterSettings();

    var sb = new StringBuilder();

    using (var xmlWriter = XmlWriter.Create(sb, xmlWriterSettings))
    {
        new XmlSerializer(typeof(string)).Serialize(xmlWriter, string.Empty);
        return sb.ToString();
    }
}
```

程式碼就是很簡單的把一個空字串拿去序列化, 但是輸出的字串卻是 `<?xml version="1.0" encoding="utf-16"?><string />`, 即使特別再手動設定 Encoding 也是一樣, 如: `var xmlWriterSettings = new XmlWriterSettings() { Encoding = Encoding.UTF8 };`

稍微推敲一下程式碼, 猜測應該是 `StringBulider` 的問題, 可能是因為 `StringBulider` 操作字串有自己的 Encoding, 所以就往這方面去找.

### 原因
因為~~今天比較懶~~不想自己往底部去鑽, 所以先問問 google 大神先, 也是運氣好還真的有找到[一篇分析文](https://blogs.msdn.microsoft.com/kaevans/2008/08/11/xmlwritersettings-encoding-being-ignored/), 大意上就是 `StringBuilder` 設計上是直接操作字元而不是 bytes, 而字串在 .NET 裡面都是 UTF-16 編碼的.

### 解法
所以這邊改用存取 Stream 的方式來操作, 如下
``` csharp
private static string DemoForSuccess()
{
    var xmlWriterSettings = new XmlWriterSettings() { Encoding = Encoding.UTF8 };

    using (var ms = new MemoryStream())
    using (var sr = new StreamReader(ms))
    using (var xmlWriter = XmlWriter.Create(ms, xmlWriterSettings))
    {
        new XmlSerializer(typeof(string)).Serialize(xmlWriter, string.Empty);
        ms.Position = 0;
        return sr.ReadToEnd();
    }
}
```

### 結論
也沒什麼特別的技巧好結論, 特別寫出來只是怕之後忘記, 再遇到還要再踩一次.

### 參考
[XmlWriterSettings Encoding Being Ignored?](https://blogs.msdn.microsoft.com/kaevans/2008/08/11/xmlwritersettings-encoding-being-ignored/)

---
