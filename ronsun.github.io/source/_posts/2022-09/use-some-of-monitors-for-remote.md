---
title: 遠端連線時支援"部份多螢幕"
date: 2022-09-25 00:16:38
categories:
- Tools
tags:
---

使用 Windows 內建的遠端連線工具 (Remote Desktop Connection) 如果需要支援雙螢幕的話需要做一些設定，這些網路上資源很多沒什麼好說的。  

但是如果是想要只使用三個螢幕中的其中兩個螢幕呢?

<!--more-->

### 手動編輯 .rdp 檔案
RDP 的圖形化介面沒有提供部分多螢幕的設定，只能將 `*.rdp` 檔案用文字編輯器打開後編輯。  

增加設定如下:  
```
use multimon:i:1
selectedmonitors:s:0,1
```

意思是支援多螢幕，然後套用到 0、1 兩個螢幕，不確定 0、1 怎麼編號的，但是要用的時候試一下就好也就不仔細追究了。  

### 結論
很空虛的一篇，但是不是太熱門的問題不想之後需要又要到處找所以還是隨手紀錄一下。  

### 參考
[remote desktop connection on 2 out of 3 monitors](https://superuser.com/a/1555631/809665)
