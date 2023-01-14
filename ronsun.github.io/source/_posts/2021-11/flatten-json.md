---
title: Flatten JSON
date: 2021-11-14 01:25:06
categories:
- C#
- .NET
tags:
---

幾年前因為需要將"不固定格式且不可預期"的 JSON 字串轉換成 `Dictionary<strin, string>` 而做了擴充方法, 但後來發現實作得太複雜了.所以想記錄一下比較簡單的做法.  

<!--more-->

### 搭配 Newtonsoft.Json
一開始的思路是將字串轉換成 `JObject` 後再迴圈所有的欄位, 根據那些欄位是 Object / Aarray / Value 搭配遞迴把整個 JSON 字串轉換成 `Dictionary<strin, string>`, 雖然遞迴有點費神但是整體也就四五十行程式碼, 寫完覺得很滿意.  

沒想到最近發現了[這一篇回答](https://stackoverflow.com/a/35838986/8223582)的精妙解法, 才發現之前的作法簡直是土法煉鋼, 所以稍微改了一下.  
> 連結中的 Code 不支援第一層是陣列的 JSON, 例如: `[{"key", "a"}, {"key", "b"}]`, 所以需要改過.

``` csharp
public static class JContainerExtensions
{
    public static Dictionary<string, string> Flatten(this JContainer jContainer)
    {
        IEnumerable<JToken> jTokens = jContainer.Descendants().Where(p => p.Count() == 0);
        return jTokens.ToDictionary(jToken => jToken.Path, jToken => jToken.ToString());
    }
}
```

不到十行而且又簡單明瞭, 呼叫端使用上也很容易:
``` csharp
var jContainer = JToken.Parse(testData) as JContainer;
var flattenJContainer = jContainer?.Flatten();
```

> `JToken.Parse(testData) as JContainer` 這一行是成立的, 會成立是因為 `JToken.Parse(testData)` 的結果可能是 `JObject` / `JArray` / `JValue`, 除了 `JValue` 外都繼承了 `JContainer`, 所以轉型上不會有問題, 而 `JValue` 是沒有 key 的純值 (例如: `JToken val = JToken.Parse("abc");`), 本來就不適合出現在要轉換成 `Dictionary<string, string>` 的情境.  

### 搭配 Sysetem.Text.Json
這個是內建的 Json 處理工具, 不過功能沒有 Newtonsoft.Json 那麼多, 沒找到比較簡單的做法所以只好還是用遞迴, 在實作上會麻煩很多.  

先來點擴充方法:  

``` csharp
public static class JsonDocumentExtensions
{
    public static Dictionary<string, string> Flatten(this JsonDocument document)
    {
        var result = new Dictionary<string, string>();
        FlattenElement(string.Empty, document.RootElement, result);
        return result;
    }

    private static void FlattenElement(string key, JsonElement element, Dictionary<string, string> result)
    {
        switch (element.ValueKind)
        {
            case JsonValueKind.Object:
                foreach (var pty in element.EnumerateObject())
                {
                    string nextKey = string.IsNullOrEmpty(key) ? pty.Name : $"{key}.{pty.Name}";
                    FlattenElement(nextKey, pty.Value, result);
                };
                break;
            case JsonValueKind.Array:
                int i = 0;
                foreach (var arrayItem in element.EnumerateArray())
                {
                    string nextKey = $"{key}[{i}]";
                    FlattenElement(nextKey, arrayItem, result);
                    i++;
                }
                break;
            default:
                result.Add(key.ToString(), element.ToString());
                break;
        }
    }
}
```

呼叫端是這樣:  
``` csharp
var flattenJsonDocument = JsonDocument.Parse(testData).Flatten();
```

### 結論
用了 Newtonsoft.Json 結果因為沒發現套件已經提供的功能而自己土法煉鋼真的沒效率又容易錯, 算是被以前的自己雷到.   

另外為了方便驗證, 有範例專案[在這裡](https://github.com/ronsun/Demo/tree/master/FlattenJson).  

### 參考
[C# flattening json structure](https://stackoverflow.com/a/35838986/8223582)