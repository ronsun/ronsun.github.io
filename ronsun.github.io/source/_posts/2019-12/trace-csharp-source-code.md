---
title: 追蹤與反組譯 C# 程式碼
date: 2019-12-29 18:26:22
categories:
- Tools
tags:
---

在開發 C# 程式或是學習過程, 會用到許多不是由我們開發的框架或套件, 很多時候會需要知道內部的運作方式, 這篇紀錄幾個方式來讓我們能知道套件內部究竟做了什麼.  

<!--more-->

### 官方原始碼
最準確的方式, 直接看官方提供的原始碼, 可能存放在 GitHub 或其他儲存庫, 一般小型第三方套件應該都能快速了解, 但是如果要看的是 .NET Framework 或是超大型套件時, 就會比較費神, 而且也無法期待所有套件都有開源.  

另外 .NET Framework 和 .NET Core 都有原始碼查詢網站, 可以快速找到原始碼的實作內容.  
+ [.NET Framework Reference Source](https://referencesource.microsoft.com/)  
+ [.NET Core Source Browser](https://source.dot.net/)

### Visual Studio 2019
直接內建在 Visual Studio 中, GoToDefinition (F12) 進入就可以看到, 設定方式也很簡單: 
`Tools > Options > Text Editor > C# > Advanced > enable navigation to decompiled sources (experimental)`  

但是這個功能在 .NET Core 的專案上無法反組譯出正確內容, 選項上也標明 experimental.  

{% asset_img vs_decompil_001.png %}   

### 其他工具
#### ILSpy
第三方反組譯工具很多, 其中我最推薦的是 ILSpy, 可在 `Extensions > Manage Extensions` 裡面找到並安裝, 能在 Visual Studio 直接按右鍵開啟 ILSpy 非常方便.  
{% asset_img ilspy_decompil_001.png %}   

開啟後也可以選擇 C# 版本  
{% asset_img ilspy_decompil_002.png %}   

另外也可以選擇 C# 跟 IL 同時顯示, 可以對照著看, 了解不同寫法編譯後是不是一樣的內容, 看不懂 IL Code 也沒關係, 滑鼠移過去還會出現說明框, 點擊後會開啟瀏覽器導頁到 msdn, 這部分在實驗學習的過程很好用.  

{% asset_img ilspy_decompil_003.png %}   

#### dotPeek 和 JustDecompile
比起 ILSpy 介面好看很多, 很多方便的功能, 如果是要仔細看程式流程的話, 這兩者更適合.  

### 結論
反組譯工具所產生的程式碼不一定跟原本的內容一樣, 如果真的要研究實作方式等細節的話, 還是直接看開源的程式碼內容比較準確.  

### 參考
[ILSpy](https://github.com/icsharpcode/ILSpy)  
[dotPeek](https://www.jetbrains.com/decompiler/)  
[JustDecompile](https://www.telerik.com/products/decompiler.aspx)  