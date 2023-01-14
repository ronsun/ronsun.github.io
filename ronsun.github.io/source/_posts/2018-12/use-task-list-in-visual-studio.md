---
title: Vsual Studio 中的工作清單(Task List)
date: 2018-12-13 01:11:40
categories:
- Tools
tags:
---

在開發的過程, 常常有一些需要改進但現階段沒時間做的部分, 我們在上面加上一個 `// TODO:` 的註解來標註, 甚至依照特性的不同會用不一樣的語彙(Token)來標註, 例如: `// BadSmell: ` 等等, Visual Studio 已經有內建幾個常用語彙, 除了這些內建語彙外我們也可以自己新增團隊內部使用的語彙.  

另一方面, 這些工作清單慢慢累積後就會變得不好管理, 這部分也可以通過 Visual Studio 提供的相關功能來協助我們管理.    

<!--more-->

### 語彙(Token)的種類
Visual Studio 內建幾種語彙與建議使用方式如下  
+ TODO: 未實作的功能.
+ HACK: 為了解決緊急問題而寫的程式, 但不是個適當的做法, 例如寫死一些變數或參數的數值先應急, 但這個數值不應該是寫死的.
+ UNDONE: 為了解決短期問題而移除或註解掉程式碼, 或是需要切換並觀察兩個不同的做法哪種比較好. 
+ UnresolvedMergeConflict: 某一小段程式碼單獨運作良好, 但放到整個專案中運作會有問題, 需要 debug 找出問題. 

> UnresolvedMergeConflict 的情境看起來比較像單元測試都能過, 但是整個流程測下來有問題, 可能是跟依賴物件的互動不正確, 或是是依賴度太高, 導致很容易被流程或是依賴物件影響他的正確性.

雖然說教學是這樣建議, 不過團隊如果有其他共識的話其實就大家說好就好, 另外, 這些語彙是不分大小寫的, 所以 TODO 和 ToDo 是一樣的, 但還是跟著團隊習慣, 盡量一致比較好.

### 工作清單視窗 (Task List Explorer)
我們可以點選 **View > Task List** 來打開工作清單視窗, 工作清單視窗中可以有篩選器或是關鍵字搜尋可以用, 很方便, 不需要用全域搜尋之類的方式找關鍵字.  

{% asset_img task-list-explorer.gif %}  


### 自定義語彙
我們可以在 **Tools > Options** 中的 **Task List** 頁籤中增加自定義的語彙, 例如我們要加入一個語彙叫做 BadSmell, 用來表示程式碼中的壞味道, 方便之後重構或整理.  

> 可以在 Task List 中改語彙的優先權(Piority), 但無法改名, 想要改名只能加上新的再刪掉舊的.  

{% asset_img task-list-add.gif %}  

### 結論
這篇也沒什麼特別難的, 主要是對於內建語彙的使用時機跟目的沒找到太多說明, 剛好有一本書有提到, 就稍微記一下, 而且自己平常很少用這個功能容易忘, 就順便備忘一下, 然後再順便試試一下用螢幕錄影轉 GIF 檔來說明操作流程的效果.

### 參考
[Mastering Web Development with Microsoft Visual Studio 2005 (P.45-P.46)](https://books.google.com.tw/books?id=XXtoCqnxXVUC&printsec=frontcover&hl=zh-TW#v=onepage&q&f=false)

---
