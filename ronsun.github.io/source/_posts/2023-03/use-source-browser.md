---
title: 使用 SourceBrowser 建立原始碼瀏覽網站
date: 2023-03-12 02:58:01
categories:
- Tools
tags:
---

[Reference Source](https://referencesource.microsoft.com/) 是一個非常方便的資源可以用來查看 .NET Framework 的程式碼，而這個網站是利用 [SourceBrowser](https://github.com/KirillOsenkov/SourceBrowser) 所建立的。  

對於我們自己開發的專案，雖然都有原始碼好像用不到這個工具，但很多時候想快速查看程式碼卻又不想耗費資源在執行 IDE 上的時候(尤其同時維護多專案的時候)，這個工具能提供一個不錯的介面來使用。

官方提供了基本的使用教學，但要開 Visual Studio 來編譯 SourceBrowser 專案，比較不方便，所以這邊會使用官方也有提供的 [NuGet Package](https://www.nuget.org/packages/SourceBrowser) 來做，稍微紀錄一下過程，讓之後需要的時候能更快的使用。

<!--more-->

### 基本使用步驟
#### 下載 NuGet Package
下載  [NuGet Package](https://www.nuget.org/packages/SourceBrowser) 後解壓縮後只留下 tools 資料夾。

#### 使用 HtmlGenerator.exe 來建立資料
根據 [GenerateTestSite.cmd](https://github.com/KirillOsenkov/SourceBrowser/blob/main/GenerateTestSite.cmd) 可以知道我們需要用 HtmlGenerator.exe 來解析目標專案並輸出到指定目錄下，如下範例：  
``` ps
tools\HtmlGenerator.exe "MyGithub\MoreNet.Cryptography\MoreNet.Cryptography.sln" /out:tools\site
```

#### 將輸出結果佈署到伺服器上
根據 [RunTestSite.cmd](https://github.com/KirillOsenkov/SourceBrowser/blob/main/RunTestSite.cmd) 可以知道我們需要使用 Microsoft.SourceBrowser.SourceIndexServer.dll 來啟動 SourceBrowser 站台 (也可以直接執行 Microsoft.SourceBrowser.SourceIndexServer.exe)，例如：
``` ps
dotnet Microsoft.SourceBrowser.SourceIndexServer.dll
```

如果是在本機執行，從瀏覽器瀏覽 http://localhost:5000 就可以看到結果了，也可以佈署到伺服器上提供線上查詢。

### 其他
+ 相關的指令可以自己寫成 `*.bat` 檔方便重複使用。
+ 產生出來的內容至少依賴 dotnet，詳細資訊見官方說明，我本機上環境太雜沒辦法精準驗證需要哪些依賴。
+ 可能還有其他使用方式和參數，但目前不急著知道所以需要的時候再看 GitHub 就好

### 結論
雖然從 GitHub 或其他線上儲存庫就能直接看程式碼，但是 SourceBrowser 提供更強大的功能來查看程式碼之間的參考與依賴關係，還是能讓我們更有效率的查看程式碼。  

唯一美中不足的是它不是靜態網站且有一些框架的依賴，所以佈署成本滿高的。

### 參考
[SourceBrowser](https://github.com/KirillOsenkov/SourceBrowser)