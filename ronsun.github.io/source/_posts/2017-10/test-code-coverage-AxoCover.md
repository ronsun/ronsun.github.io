---
title: 在Visual studio 2017上使用AxoCover顯示測試程式碼涵蓋範圍
date: 2017-10-09 20:57:38
categories:
- Tools

---

當我們想知道目前產品的測試程式涵蓋率的時候, 可以使用Visual Studio 2017內建的功能來分析, 但只有企業版支援這個功能, 所以另外找了AxoCover來用看看。

<!--more-->

### AxoCover

#### 安裝
打開 Tools > Extensions and Updates, 並找到AxoCover後直接安裝(安裝時需要關閉Visual Studio才能繼續)。 
{% asset_img AxoCover-Install.png %}


#### 使用
安裝完成開啟Visual Studio後, 可在Tool > AxoCover打開功能視窗。  
{% asset_img AxoCover-Installed.png %}   

一開始的功能視窗看起來什麼都沒有, 照著上面的提示編譯一下專案後, 可以看到完整的功能頁面。  
{% asset_img AxoCover-Function-Window.png %}

接著就幾個主要功能說明一下  

+ **Tests**  
  - **Tests > Run**  
    執行所有單元測試。  

  - **Tests > Cover**  
    分析測試涵蓋率, 結果可在Report頁籤看到。  

  - **Tests > Build**  
    編譯。 

+ **Reposrt**  
  這邊可以看到涵蓋率的分析。  
  - **Reposrt > Export**  
    將執行與涵蓋率分析結果分別輸出到`~/.axoCover/runs` 以及`~/.axoCover/reports`, 其中`~/.axoCover/reports`下是用一個精美的靜態網頁來顯示測試涵蓋率報表。  
     {% asset_img AxoCover-Report.png %}

+ Settings
  AxoCover的相關設定全在這。  


操作上大致是這樣子的, 接著可以打開程式碼並且清楚的看到有被測試碼涵蓋到的部分是綠色, 沒涵蓋到的部分是紅色, 而部分涵蓋的那一行會有一個很小的黃色圖標。  
{% asset_img AxoCover-Display.png %}
