---
title: XmlSerializer 的序列化/反序列化 與 BOM
date: 2018-12-03 16:02:39
categories:
- C#
- .NET
tags:
---

XmlSerializer 提供許多序列化和反序列化相關的多載, 都有各自的優缺點, 例如有些方式在某些編碼下會因為 BOM(byte order mark) 出現在開頭而導致反序列化出現 Exception 或是序列化出含有 BOM 的字串導致接收端無法成功解析, 而各種方式的彈性與易用性都略有不同, 所以稍微記錄幾種序列化和反序列化的方式與特徵.

<!--more-->

### 序列化

#### XmlWriter
這種做法最有彈性, 可以靠著改變 XmlWriterSettings 以及 XmlSerializerNamespaces 的設定去控制序列化出來的 XML 字串的格式與內容, 但要特別注意的是 XmlWriterSettings 預設是有 BOM 的 UTF-8 編碼, 如果接收資料的一方無法處理這種格式的話, 需要特別將 XmlWriterSettings.Encoding 設成無 BOM 的編碼格式, 例如: `var ws = new XmlWriterSettings() { Encoding = new UTF8Encoding() }`, 比較適合用在需要跟許多不同外部系統介接的模組上

``` csharp
public string Serialize(Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(MyModel));

	var ws = new XmlWriterSettings()
	{
		Encoding = encoding
	};

	var ns = new XmlSerializerNamespaces(new XmlQualifiedName[] { XmlQualifiedName.Empty });

	using (var ms = new MemoryStream())
	using (var xw = XmlWriter.Create(ms, ws))
	{
		serializer.Serialize(xw, new MyModel(), ns);
		return ws.Encoding.GetString(ms.ToArray());
	}
}
```

#### StreamWriter
這種做法彈性小了一點, 只能用 XmlSerializerNamespaces 來控制 namespace, 比較難透過設定參數來改變輸出格式和內容, 但本例還是可以透過改變參數 encoding 自己決定編碼格式, 且另外一個建構子多載 `StreamWriter(Stream)` 預設就是無 BOM 的 UTF-8, 比起使用 XmlWriter 能更直覺的避免序列化時產生 BOM.

``` csharp
public string Serialize(Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(MyModel));

	var ns = new XmlSerializerNamespaces(new XmlQualifiedName[] { XmlQualifiedName.Empty });

	using (var ms = new MemoryStream())
	using (var sw = new StreamWriter(ms, encoding)) // 2nd arg default to utf-8 no BOM
	{
		serializer.Serialize(sw, new MyModel(), ns);
		return encoding.GetString(ms.ToArray());
	}
}
```

#### Stream
這種做法彈性就更小了, 只能用 XmlSerializerNamespaces 來控制 namespace, 除了比較難透過設定參數來改變輸出格式和內容外, 參數 `ecoding` 也只能是 UTF-8, 但另一方面, 程式碼比較簡短且序列化出來的 XML 字串不會有 BOM, 用在不需要跟外部系統介接的模組上非常適合.

``` csharp
public string Serialize(Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(MyModel));

	var ns = new XmlSerializerNamespaces(new XmlQualifiedName[] { XmlQualifiedName.Empty });

	using (var ms = new MemoryStream())
	{
		serializer.Serialize(ms, new MyModel(), ns);
		return encoding.GetString(ms.ToArray());
	}
}
```

#### StringWriter
這種跟直接使用 Stream 的做法差不多都不容易改輸出格式和內容, 優點也一樣是簡短清晰且不用考慮 BOM 的問題, 而用 StringBuilder 比上面用 Stream 多了一個特性是連編碼格式都不用考慮, 同樣適合用在不需要跟外部系統介接的模組上. 

> StringBuilder 預設的編碼格式是 UTF-16

``` csharp
public string Serialize()
{
	var serializer = new XmlSerializer(typeof(MyModel));

	var ns = new XmlSerializerNamespaces(new XmlQualifiedName[] { XmlQualifiedName.Empty });

	var sb = new StringBuilder();
	using (var sw = new StringWriter(sb))
	{
		serializer.Serialize(sw, new MyModel(), ns);
		return sb.ToString();
	}
}
```

#### 小結
每種做法都有各自的優缺點, 需要在輸出格式上滿足各種情境的話, 用 XmlWriter 操作是最適合的, 如果編碼與輸出格式單一且幾乎不會改變的話, 用 Stream 或 StringWriter 處理會更好懂. 如果使用 XmlWriter 或是 SreamWriter 兩個範例操作的話要注意序列化出來的內容是否有 BOM, 尤其是需要跟外部系統介接時.

### 反序列化
#### XmlReader
XmlReader 是提供最多參數的方式, `XmlReader.Create(Stream, XmlReaderSettings)` 和 XmlReader.Create(TextReader, XmlReaderSettings) 的第二個參數有很多選項可以調整, 從方法定義可以看得出來 XmlReader.Create(...) 有很多多載可以用, 如果是從 Stream 讀取資料的話, 即使 XML 字串有 BOM 也不會出錯, 但如果從 StringReader 讀取, 在 XML 字串有 BOM 的情境下會拋出 Exception.  

``` csharp
public T DeSerialize_FromStream<T>(string xml, Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(T));

	var xmlBytes = encoding.GetBytes(xml);
	using (var ms = new MemoryStream(xmlBytes))
	using (var xr = XmlReader.Create(ms))
	{
		xr.Read();
		var result = serializer.Deserialize(xr);
		if (result != null)
		{
			return (T)result;
		}
		return default(T);
	}
}

public T DeSerialize_FromStringReader<T>(string xml)
{
	var serializer = new XmlSerializer(typeof(T));

	using (var sr = new StringReader(xml))
	using (var xr = XmlReader.Create(sr))
	{
		xr.Read();
		var result = serializer.Deserialize(xr);
		if (result != null)
		{
			return (T)result;
		}
		return default(T);
	}
}

```

#### StreamReader / Stream
這兩種方式差不多, 即使 XML 字串有 BOM 也都不會出錯, 只差在 StreamReader 建構子的參數比較多, 如果對編碼格式沒特別的需求的話, 直接操作 Stream 比較單純.

``` csharp
public T DeSerialize_StreamReader<T>(string xml, Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(T));

	var xmlBytes = encoding.GetBytes(xml);
	using (var ms = new MemoryStream(xmlBytes))
	using (var tr = new StreamReader(ms, encoding))
	{
		var result = serializer.Deserialize(tr);
		if (result != null)
		{
			return (T)result;
		}
		return default(T);
	}
}

public T DeSerialize_Stream<T>(string xml, Encoding encoding)
{
	var serializer = new XmlSerializer(typeof(T));

	var xmlBytes = encoding.GetBytes(xml);
	using (var ms = new MemoryStream(xmlBytes))
	{
		var result = serializer.Deserialize(ms);
		if (result != null)
		{
			return (T)result;
		}
		return default(T);
	}
}
```

#### StringReader
這種方法在 XML 字串有 DOM 的時候會出 Exception, 優點是不用考慮編碼格式, 只需要知道 XML 字串就可以了. 

``` csharp
public T DeSerialize<T>(string xml)
{
	var serializer = new XmlSerializer(typeof(T));

	using (var sr = new StringReader(xml))
	{
		var result = serializer.Deserialize(sr);
		if (result != null)
		{
			return (T)result;
		}
		return default(T);
	}
}
```

#### 小結
反序列化大部分的做法都很簡單, 以目前實驗看來, 把 XML 字串先轉成 Stream 再處理都可以相容有 BOM 的資料來源, 需要考量的點似乎也比序列化時少很多.

### 結論
這篇只是要實驗各種序列化與反序列化的方式遇到 BOM 時候的結果, 所以沒有特別實驗各種不同方式在其他複雜的 XML 資料下的轉換結果, 如果之後有遇到其他問題的話再回來補.  

範例程式碼可參考 [Demo](https://github.com/ronsun/Demo/tree/master/XmlSerializerAndBOM)