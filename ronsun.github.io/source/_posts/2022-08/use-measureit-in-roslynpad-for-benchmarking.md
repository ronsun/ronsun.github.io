---
title: 在 RoslynPad 中使用 MeasureIt 做基準測試 (benchmarking)
date: 2022-08-18 23:42:06
categories:
- Tools
tags:
---

BenchmarkDotNet 是一個很熱門的工具可以用來簡單的量測與比較程式碼的效能，但是有時候我們只是想要快速測試一段程式碼的執行時間，這時候還要開 IDE 使用 BenchmarkDotNet 就覺得有點厚重 (除非 BenchmarkDotNet 專案已經建好在方案中可以一邊開發一邊實驗)。  

這時候就想要在 RoslynPad 或 LinqPad 中使用基準測試工具，偏偏因為 RoslynPad 的限制而無法支援 BenchmarkDotNet，所以只能另外找一個替代方案 - MeasureIt。

<!--more-->

### 使用方式
#### 範例
```
#r "nuget:MeasureIt.exe/0.2.2"

using PerformanceMeasurement;

var cost = LinqPadUX.Measure.Action(new Action(() =>
{
    string s = string.Empty;
    for (var i = 0; i < 1000; i++)
    {
        s += i.ToString();
    }
})).Dump();

var costComparison = LinqPadUX.Measure.NamedActions(new List<NamedAction>()
{
    new NamedAction("string", () =>
    {
        string s = string.Empty;
        for (var i = 0; i < 1000; i++)
        {
            s += i.ToString();
        }
    }),
    new NamedAction("stringBuilder", () =>
    {
        var sb = new StringBuilder();
        for (var i = 0; i < 1000; i++)
        {
            sb.Append(i.ToString());
        }
        var s = sb.ToString();
    })
}).Dump();
```

### 結論
沒什麼難度，但是沒記一下的話很快就忘了，尤其這個替代方案不算熱門，沒有很好找。  

這個工具的功能比較精簡，如果真的要複雜的分析還是 BenchmarkDotNet 比較豐富。

### 參考
[Support BenchmarkDotNet](https://github.com/roslynpad/roslynpad/issues/118#issuecomment-403788063)  

[Run time costs of small operations in C#](http://ig2600.blogspot.com/2012/12/run-time-costs-of-small-operations-in-c.html)  