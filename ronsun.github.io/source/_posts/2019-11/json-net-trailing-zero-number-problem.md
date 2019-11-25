---
title: Json.NET 反序列化數字時, 尾數 0 可能被捨棄
date: 2019-11-26 00:09:43
categories:
- C#
- Packages
tags:
---

用 Json.NET 來將 JSON 字串反序列化成物件是很典型的做法, 以前也沒出過什麼問題, 但在某些使用方式下, 會出現數字的尾數 0 被移除的狀況, 例如: 10.00 會被轉成 10, 而 10.10 會被轉成 10.1.  

雖然這個結果乍看之下沒什麼影響, 但如果資料是在跟第三方 API 介接時, 簽章 (sign) 所需要的欄位時, 就會直接導致簽章檢核錯誤.  

<!--more-->

### 正常情境
以我之前習慣將 JSON 字串反序列化成 Model 的用法來說, 是不會有這個問題的, 如下.  

``` csharp
public class Data
{
    [JsonProperty("money")]
    public string Money { get; set; }
}

static void Main(string[] args)
{
    string jsonString = "{\"money\":10.10}";
    var data = JsonConvert.DeserializeObject<Data>(jsonString);
    // output: 10.10
    Console.WriteLine(data.Money);
}
```

### 出問題的情境
#### 問題描述
如果我們的 API 接受大量來自外部的呼叫, 且 JSON 的欄位名稱與結構不固定, 我們可能不希望建立大量的 Model 來反序列化, 那程式碼可能會變成下面這樣: 

``` csharp
string jsonString = "{\"money\":10.10}";
var data = JObject.Parse(jsonString);
// output: 10.1
Console.WriteLine(data.GetValue("money"));
```

> 不只是直接用 `JObject` 取值的情境, 要把 JSON 字串轉成一個 `Dictionary` 或類似情境時, 都會碰到這個問題.  

#### 解決方式
雖然複雜很多, 但用在解析不固定結構的 JSON 字串 (`jsonString`) 並取出特定欄位或所有欄位時很方便, 且要取出的欄位名稱可做為參數也支援巢狀的結構(只要符合 JSONPath expression 的規則).  

``` csharp
string jsonString = "{\"money\":10.10}";
using (var sr = new StringReader(jsonString))
using (var jr = new JsonTextReader(sr))
{
    jr.FloatParseHandling = FloatParseHandling.Decimal;
    var jsonContent = JToken.ReadFrom(jr);
    var target = jsonContent.SelectToken("money");
    // output: 10.10
    Console.WriteLine(target);
}
```

### 結論
一般情境下, 反序列化還是建議盡量轉換成 Model, 後續會好操作很多, 且可以減少用 `JObject` 等方式操作造成執行階段容易出錯的疑慮. 但在現實專案中, 條件總是比較複雜, 還要考慮原本系統的設計與現況, 所以還是可能會遇到不適合反序列化成特定 Model 的情境, 就需要靈活變化了.  