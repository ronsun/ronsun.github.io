---
title: 使用 Friend Assembly
date: 2021-06-06 00:12:57
categories:
- C#
- .NET
tags:
---

如果組件 A 是組件 B 的 Friend Assembly, 則在組件 A 中能存取別的組件內的 `internal` 類別或成員, 更多細節在[官方介紹](https://docs.microsoft.com/en-us/dotnet/standard/assembly/friend).  

<!--more-->

### 用法
加上一行 `[assembly: InternalsVisibleTo("AssemblyName")]` 就好, 建議放在 Properties 資料夾下的 AssemblyInfo.cs 檔案中.

### 使用情境
目前有想到適合的情境是
1. **跨組件的內部 API**: 拆分成好幾個組件組合成一個完整產品, 希望給在這個產品中讓特定組件能跨組件存取, 但又不希望用 `plubic` .  
2. **測試目的**: 因為測試的目的, 需要開放給測試專案存取.  

### 結論
使用上很簡單, 但是比較需要想的是試用在甚麼情境, 目前也就為了給測試專案存取而使用有用到而已.  

### 參考
[Friend assemblies](https://docs.microsoft.com/en-us/dotnet/standard/assembly/friend)