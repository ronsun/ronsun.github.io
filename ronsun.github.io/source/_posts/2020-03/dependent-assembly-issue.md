---
title: 套件相依性問題
date: 2020-03-01 18:48:21
categories:
- Others
tags:
---

這個問題比較複雜, 一開始引發問題的情境是  

> A 專案依賴 B 專案, 且雙方都依賴同一個套件, 但卻是不同版本的套件

由於情境比較複雜, 所以我開了一個[範例專案 DependentAssemblyIssue](https://github.com/ronsun/Demo/tree/master/DependentAssemblyIssue) 來做示範, 內文會依照情境來描述處理方式以及所造成的後遺症.  

<!--more-->

### 初始可運作的情境

以範例專案中, 第一個 commit `1c5d263` 為例, 依賴關係如下:  
+ Client.MVC 依賴 Library
+ Client.MVC 依賴 Newtonsoft.Json 6.0.1 版
+ Library 依賴 Newtonsoft.Json 6.0.1 版

其中在 Client.MVC 專案的 `Global.asax.cs` 中呼叫 Library 專案的靜態方法 `Worker.Do()` 如下:  
``` csharp
protected void Application_Start()
{
    AreaRegistration.RegisterAllAreas();
    FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
    RouteConfig.RegisterRoutes(RouteTable.Routes);
    BundleConfig.RegisterBundles(BundleTable.Bundles);

    Worker.Do();
}
```

`Worker.Do()` 會在 Client.MVC 開啟後在 Visual Studio 的 Output 視窗印一行 log, 可以正常運作.  

### 升版 Library 依賴的 Newtonsoft.Json 引發錯誤
接下來我們假設維護 Library 專案的小組將 Newtonsoft.Json 的版本升級到 8.0.1 (commit `89d3eb9`), 這時候 Client.MVC 還是依賴 6.0.1 版, 而當其呼叫 `Worker.Do()` 時, 就會拋出例外
```
System.IO.FileLoadException: 'Could not load file or assembly 'Newtonsoft.Json, Version=8.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed' or one of its dependencies. The located assembly's manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)'
```

這是因為 Client.MVC 在編譯期間根據 `Web.config` 中的設定將對版本 0.0.0.0 到 6.0.0.0 (oldVersion) 的參考導向 6.0.0 (newVersion) 但是 Library 專案依賴的是 8 版的關係.  
``` xml
<dependentAssembly>
  <assemblyIdentity name="Newtonsoft.Json" publicKeyToken="30ad4fe6b2a6aeed" />
  <bindingRedirect oldVersion="0.0.0.0-6.0.0.0" newVersion="6.0.0.0" />
</dependentAssembly>
```

### 解決方式
解決方式有兩個方向, 但是都有相對的副作用.  

#### 修改 oldVersion
將 `bindingRedirect` 的屬性 `oldVersion` 改成 `0.0.0.0-8.0.0.0`, 將 8 版以下的 Newtonsoft.Json 依賴繫結到 6 版, 如 commit `b827286` 或下方所示.  
``` xml
<bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="6.0.0.0" />
```

但是這會有個嚴重的副作用, 由於編譯出來後是使用 6 版的套件, 當 Library 專案中呼叫了 8 版的套件的新 API 時, 會發生編譯時期甚至單元測試都沒問題, 但是卻在 Client.MVC 的執行階段發生錯誤, 如 commit `2cc0617` 所示.  

另一個更難除錯的情境的是, 當所使用的 API 接口沒有改變, 但是改變了實作細節導致新舊版本執行結果不同的時候, 可能連錯誤訊息都沒有就更難除錯了(這部分沒找到適合展示與驗證的 API, 只是推測).  

#### 同時升版 Client.MVC 依賴的 Newtonsoft.Json
這就像 commit `4ec7c50` 這樣, 直接將 Client.MVC 所依賴的套件版本升到跟 Library 相同版本, 另外 `bindingRedirect` 也要修改如下.  
``` xml
<bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
```

但這也意味了 Library 所依賴的套件版本更改時, Client.MVC 也需要修改相應的套件版本, 這在維護上是很容易忽略的細節, 而如果 Library 和 Client.MVC 是兩個不同的小組維護的時候, 這個問題就更容易延後發現了.  

除了維護不方便的問題外, Client.MVC 修改相應的套件版本也就意味著他直接使用該套件的程式碼都需要重新測試, 避免新版本的 API 與舊版本運作結果不同.  

### 結論
這個問題發生的時候其實花了不少時間去找, 因為對於 Client.MVC 來說是明明什麼都沒做但他就壞了, `Web.config` 中的設定是一開始引入套件時就自動加好的, 所以也沒有想到有這一段.  

事後其實有想過怎麼在未來預防這類的問題, 目前來說還沒想到如何簡單又漂亮的完全避開這個問題, 一個想法就是盡量避免這樣的依賴關係, 從簡化專案與套件的依賴這個角度可以有一定程度的效果, 但無法完全避免, 因為從現實專案來看就真的兩個專案都要直接依賴 Newtonsoft.Json, 而特別幫 Newtonsoft.Json 做一層隔離層讓兩個專案依賴也是一種方式, 但如果類推到每個套件都要做公用的隔離層那改動就比較大了.  

### 參考
[Redirecting Assembly Versions](https://docs.microsoft.com/en-us/dotnet/framework/configure-apps/redirect-assembly-versions#redirecting-assembly-versions-by-using-publisher-policy)  

[Understanding a csproj assembly reference](https://stackoverflow.com/questions/16578819/understanding-a-csproj-assembly-reference/16580870)  
