---
title: 為什麼陣列索引的存取是常數時間複雜度 O(1)
date: 2024-04-27 22:37:10
categories:
- Others
tags:
---

可能只有我不知道，但是不記下來又覺得需要解釋的時候會解釋不出來。

<!--more-->

陣列是在程式設計中常見的資料結構，將多個**固定大小**的元素存放在**連續的記憶體位址**中，而這段記憶體的**起始位址是已知的**。當要存取陣列中的某個元素時就能透過起始位址加上偏移量來計算出目標元素的記憶體位址，而不需要一個一個的走訪所有元素去找出目標元素。  

> 另外，[Why is indexing faster than binary search](https://yinwang0.wordpress.com/2013/04/02/indexing) 這篇文章提到這件事背後與硬體運作有關。

計算目標元素的記憶體位置公式如下：  
```
元素位址 = 起始位址 + (目標索引 × 元素大小)
```

由於計算元素位址的過程不依賴於陣列的大小，而僅涉及一次計算操作，因此無論索引大小如何，這個操作的時間都是固定的。這就是為什麼說陣列索引的存取時間複雜度是 O(1)，即常數時間複雜度。  

### 結論
這是個很小的知識點，寫出來比較像是備忘並幫助我理清思緒，而需要回答這個問題時也能比較順暢的回答。 另一方面參考資料中有很多圖文並茂且深入的講解，透過連結記錄起來需要時也能更深入的了解。

### 參考
[How accessing an array element is constant time ?](https://www.linkedin.com/pulse/how-accessing-array-element-constant-time-prabaharan-balaji-zkj2c/)  
[Why does accessing an Array element take O(1) time?](https://www.geeksforgeeks.org/why-does-accessing-an-array-element-take-o1-time/)  
[Why is Array Access Constant Time](http://xahlee.info/comp/why_is_array_access_constant_time.html)  
[Why is indexing faster than binary search](https://yinwang0.wordpress.com/2013/04/02/indexing/)  