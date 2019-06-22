---
title: .NET core 測試專案版本不符
date: 2019-06-22 23:38:27
categories:
- C#
- .NET Core
tags:
---

前陣子的新專案用的是 .NET core 2.1, 在加測試專案的時候發現測試專案編譯無法通過, 拋出類似於版本衝突/版本錯誤之類的錯誤 (錯誤碼忘了, 不知道是不是 CS1705), 經過檢查後發現產品專案和測試專案的相關組件版本都一樣, 一時間還真的找不到方向.  

<!--more-->

不過很快的就靠谷歌大神找到了答案, 雖然一般來說這麼容易找到的答案實在不用記, 又遇到就再找就好, 不過這個情境比較特殊, 萬一再遇到沒下好關鍵字說不定就找不到了, 想想還是速記一下好了.  

### 解法
總之就是手動編輯 csproj 檔, 在 ItemGroup 裡面加上一段引用如下
```
<PackageReference Include="Microsoft.AspNetCore.App" />
```

### 結論
解決這個問題我自己是完全沒分析什麼, 就完全是剛好找到相關的文章, 而且正好命中問題一下就解決了, 神奇的是, 事後要在家裡重現問題都無法重現, 也不知道是不是特殊條件才會觸發這個問題, 就這樣留下一個謎...

### 參考
[[UnitTest] ASP.NET Core 2.2 測試專案中的版本衝突](http://marcus116.blogspot.com/2019/03/version-conflicts-in-test-project-on-aspnetcore-2.2.html)  
[Version conflicts in test project depending on a Microsoft.AspNetCore.App project](https://github.com/dotnet/sdk/issues/2253)  