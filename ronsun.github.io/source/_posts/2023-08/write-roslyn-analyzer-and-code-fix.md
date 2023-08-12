---
title: 客製 .NET 程式碼分析與自動修正
date: 2023-08-12 23:21:55
categories:
- Others
tags:
---

[.NET 編譯器平台 (Roslyn) 分析器](https://learn.microsoft.com/en-us/visualstudio/code-quality/roslyn-analyzers-overview?view=vs-2022) 提供很多有用的程式碼分析規則讓我們更容易維持良好的程式碼品質，但有時候我們會需要自己定義一些規則來符合專案開發規範。官方以及其他網路文章其實已經提供了不少資訊，但因為步驟和說明真的很多，了解起來太花時間，於是想自己來整理一下讓需要的人可以快速走完所有步驟。

<!--more-->

### 用 Visual Studio 樣板建立專案
1. 打開 Visual Studio (這邊用 2022 版來示範)。
2. File > New > Project，選擇 Analyzer with code fix (.NET Standard) 如下圖：  
    {% asset_img 1-create-solution-1.png %}   

    > 找不到的話就是沒安裝相關元件，[參考官方說明來安裝](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/tutorials/how-to-write-csharp-analyzer-code-fix#installation-instructions---visual-studio-installer)。  

3. 下一步繼續設定名稱路徑等，這邊要注意的是 **Framework 選單沒有意義，不管選哪一個建立的方案都是一樣的版本，所以不用費心挑選**。
    {% asset_img 1-create-solution-2.png %}   

4. 建立完成如下圖，會得到四個專案，[詳情參考官方介紹](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/tutorials/how-to-write-csharp-analyzer-code-fix#explore-the-analyzer-template)。  
    {% asset_img 1-create-solution-3.png %}   

### 建立規則並發佈
#### 開發與測試
從剛剛建立的方案所提供的範本為起點。  
1. `MyRoslynAnalyzer` 專案中寫的是主要的程式碼分析規則。
2. `MyRoslynAnalyzer.Test` 專案中寫的是測試。
3. `MyRoslynAnalyzer.Package` 提供一些製作 NuGet Package 時候所需要的設定與工具。
    > 預設就可以用不需要改。

4. `MyRoslynAnalyzer.Vsix` 設定為起始專案啟動後會另外開一個 Visual Studio 來套用前面寫的規則。
    > 我會刪掉這個專案。  
    > 他也是預設就可以用，但會多開一個 Visual Studio，這樣交錯操作對我來說容易搞混，不如單元測試完直接把 NuGet Package 放在自己本機然後用固定的專案測試。但是這個做法是因為我已經建好本機 NuGet 儲存庫了，建立本機 NuGet 儲存庫的方法之前有寫過一篇 {% post_link local-nuget-package-repository %}。  

#### 開發建議
+ **Hello World 從 [官方提供的範例專案](https://github.com/dotnet/samples/tree/main/csharp/roslyn-sdk/Tutorials/MakeConst) 開始看。**  
+ **綜合參考 [StyleCopAnalyzers](https://github.com/DotNetAnalyzers/StyleCopAnalyzers) 和 [roslyn-analyzers](https://github.com/buyaa-n/roslyn-analyzers) 來進一步學習。**  
+ **資料夾結構與命名等參考 StyleCopAnalyzers。**這是因為他用編號命名，資料夾結構用規則分類來做，整體專案也比較單純，這對維護來說很方便。  
+ **實作細節參考 roslyn-analyzers。**這是因為 StyleCopAnalyzers 畢竟是另外一套工具，他有一些自定義的內容，相較之下參考 roslyn-analyzers 的實作能最大限度的利用內建的物件避免自造輪子或額外依賴另一個套件的物件增加沒必要的複雜度。但是 StyleCopAnalyzers 的資料夾複雜，要找到可以參考的規則還真不太好找，但因為所有規則都會繼承 `DiagnosticAnalyzer` 類別並加上 `DiagnosticAnalyzerAttribute`，以這個為進入點就會好找很多。  
+ **從最簡單的命名規則開始**，會比較容易建立信心。  
+ **善用 ChatGPT 或 GitHub Copilot 輔助。**AI 在這類很常見的情境上表現得很好，我自己就只用 AI 加預設範例就實作出一些簡單的規則了。  

#### 發佈
發佈的話就是直接編譯 `MyRoslynAnalyzer.Package` 並用 Visual Studio 內建的功能來製作 nupkg 檔就好，很方便。
    {% asset_img 3-packing-1.png %}   

### 結論
其實入門不難，但是就是官方文件非常長所以要看很久。  

### 參考
[Overview of source code analysis](https://learn.microsoft.com/en-us/visualstudio/code-quality/roslyn-analyzers-overview?view=vs-2022)  

[How to write a Roslyn Analyzer](https://devblogs.microsoft.com/dotnet/how-to-write-a-roslyn-analyzer/)

[Roslyn Analyzer Part 1 - 5](https://www.alwaysdeveloping.net/p/analyzer-explained/)  

[roslyn-analyzers](https://github.com/buyaa-n/roslyn-analyzers)  

[StyleCopAnalyzers](https://github.com/DotNetAnalyzers/StyleCopAnalyzers)  
