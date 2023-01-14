---
title: Encoding.UTF8 vs new UTF8Encoding() 和 xml 序列化的可能問題
date: 2021-05-10 01:36:01
categories:
- C#
- .NET Core
tags:
---

`Encoding.UTF8` 和 `new UTF8Encoding()` 所建立的物件內容是極其相似的, 甚至有些問答網站是說完全一樣, 但實際上有個小小不同的地方, 雖然多數時候不會造成影響, 但前陣子在寫 xml 序列化的測試的時候就出現結果不同的情境, 所以稍微紀錄一下.   

<!--more-->

### 問題點

我們用下面的範例來做實驗 (示範用的, 忽略 clean code 的要求), 範例是基於 .Net Core 3.1.  

``` csharp
    public class FakeClass
    {
        public string FakeString { get; set; } = "A";
    }

    class Program
    {
        static void Main(string[] args)
        {
            var serializer = new XmlSerializer(typeof(FakeClass));
            var obj = new FakeClass();
            var xmlWriterSettings = new XmlWriterSettings() { Encoding = Encoding.UTF8 };
            var b = Serialize(serializer, xmlWriterSettings, obj);

            var xmlWriterSettings2 = new XmlWriterSettings() { Encoding = new UTF8Encoding() };
            var b2 = Serialize(serializer, xmlWriterSettings2, obj);
        }

        static byte[] Serialize<T>(XmlSerializer serializer, XmlWriterSettings xmlWriterSettings, T obj)
        {
            var ms = new MemoryStream();
            using (var xmlWriter = XmlWriter.Create(ms, xmlWriterSettings))
            {
                serializer.Serialize(xmlWriter, obj);
                return ms.ToArray();
            }
        }
    }
```

從上面的範例可以發現, `b` 和 `b2` 的內容是不一樣的, 其中 `b` (使用`Encoding.UTF8`) 的最前面多了 BOM (三個 bytes, 內容就是 `Encoding.UTF8.GetPreamble()`), 雖然看起來沒有問題, 但是如同 {% post_link xml-serialize-deserialize %} 這一篇提到的, 如果合作對象無法反序列化含 BOM 的 xml 時, 在資料交換過程就會有很多阻礙, 且含 BOM 的 XML 在文字編輯器上是無法用肉眼看出來的 (除非有特別設定顯示特殊字元).   

另外一件有趣的事, 只要 bytes 內容一樣, 兩種方式轉成字串得到的內容是相同的, 亦即 `Encoding.UTF8.GetString(b) == new UTF8Encoding().GetString(b)` 且 `Encoding.UTF8.GetString(b2) == new UTF8Encoding().GetString(b2)`.  

相關內容之前那篇提過, 這篇主要是要稍微將範圍縮小到 **Encoding.UTF8 和 new UTF8Encoding() 不完全一樣** 這件事上.  

### 看看原始碼
#### .Net Core
那既然從執行結果知道這兩者不同, 那就令人好奇這兩者的差別到底有多大了, 這時候就需要看看原始碼長怎樣了.  

先從 `Encoding.UTF8` 開始, 從 [.NET Source Browser](https://source.dot.net/#System.Private.CoreLib/Encoding.cs,a10eb90a3d884500) 往下追蹤, 可以發現最後使用的是呼叫  `UTF8Encoding(bool encoderShouldEmitUTF8Identifier)` 這個建構子, 參數帶的是 `true`, 如下:  
``` csharp
public UTF8Encoding(bool encoderShouldEmitUTF8Identifier) :
	this()
{
	_emitUTF8Identifier = encoderShouldEmitUTF8Identifier;
}
```

然後是 `new UTF8Encoding()`, [.NET Source Browser](https://source.dot.net/#System.Private.CoreLib/UTF8Encoding.cs,21efe420a875356a,references) 往下追蹤, 可以發現呼叫的是 `UTF8Encoding()` 這個建構子, 如下:  
``` csharp
public UTF8Encoding() :
	base(UTF8_CODEPAGE)
{
}
```

可以看到這兩個情境只差別會不會將 `_emitUTF8Identifier` 設為 `true` 而已.  

#### .Net Framework
在 .Net Framework 上更簡單, 如下兩段對比一目瞭然
[Encoding.UTF8](https://referencesource.microsoft.com/#mscorlib/system/text/encoding.cs,a10eb90a3d884500) 對比 [new UTF8Encoding()](https://referencesource.microsoft.com/#mscorlib/system/text/utf8encoding.cs,21efe420a875356a,references).  

### 結論
這個差別很隱諱, 而且使用起來雖然大多情境都沒問題, 但遇到問題時容易卡住很久找不到主因, 尤其是跟外部廠商交互過程遇到這種問題真的要靠大量溝通加一點通靈能力才能找到.  

另外如果沒看錯的話, UTF 系列的只有 UTF8 會有這個小差異, UTF7 和 UTF32 使用 Encoding.UTF7 / Encoding.UTF32 都各自和直接透過 UTF7Encoding / UTF32Encoding 無參數建構子得到的實例 (instance) 是一樣的.  

### 參考
[.NET Source Browser](https://source.dot.net/)  