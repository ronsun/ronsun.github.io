---
title: 資料查詢很慢的原因與因應方式
date: 2024-05-26 23:30:08
categories:
- Others
tags:
---

在實際專案中，資料查詢緩慢是一個常見的問題，如何找出具體原因成了一個很重要的主題。但在工作中，很多人往往過於注重單一面向而沒有精確的找出真正的瓶頸。例如，因為 ORM 使用不當而導致效能低落，卻將問題歸咎於 ORM 本身進而禁用 ORM。直接撰寫 SQL 查詢雖然可能較快，但失去了 ORM 的優勢。如果能先意識到 ORM 使用不當的問題並加以改善，或許能同時達到效能需求並保有 ORM 的優勢。  

找到問題後的解決方案又是另一個複雜的過程，本文將簡單記錄幾個排查資料查詢緩慢的常見方向及應對方式。

<!--more-->

### 從 ORM 開始
接續一開始說的，從 ORM 常見問題開始排查。

#### N+1 Problem (N+1 Queries)
以下面程式碼為範例，並有以下前提：
1. 使用 Entity Framework Core 作為 ORM。
2. 啟用 Lazy Loading。

``` csharp
using (var context = new MyDbContext())
{
    // 1 Query.
    var authors = context.Authors.ToList();
    foreach (var author in authors)
    {
        // Additional N Queries.
        var books = author.Books;
    }
}
```

上面的範例中總共會產生 1 (`context.Authors`) + N (`author.Books`) 次查詢。也就是說如果資料庫中有 1000 個作者，就會先執行一次查詢來查出所有作者資訊，並**在迴圈中額外查詢 1000 次各個作者的著作**。  

而正確的做法應該如下：
``` csharp
using (var context = new AppDbContext())
{
    // A single query.
    var authorsWithBooks = context.Authors
                                  .Include(a => a.Books)
                                  .ToList();
    foreach (var author in authorsWithBooks)
    {
        // No additional queries are executed here.
        var books = author.Books;
    }
}
```
這個做法在一開始查詢出作者時，也將所有作者的著作查詢出來，迴圈中就只是存取記憶體中的資料，如果資料庫中有 1000 個作者，相當於省下了 1000 次額外查詢。

#### 濫用 Lazy Loading 
如果沒有注意 Lazy Loading 的特性就很容易在存取 Model 的過程，不經意的造成大量的額外查詢。例如下面範例：

``` csharp
using (var context = new MyDbContext())
{
    // The 1st query.
    var author = context.Authors.FirstOrDefault();

    // The 2nd query.
    var books = author.Books;

    // The 3rd query
    var awards = author.Awards;

    // Much more queries via navigations.
    // ...
}
```

> Entity Framework Core 預設不啟用 Lazy Loading 就讓我覺得很放心，即使是資淺同事也不容易誤用。

#### LINQ 查詢所翻譯出來的 SQL 不夠理想
既然 ORM 的功能之一是將 LINQ 翻譯成 SQL，就應該適時的開啟相應的 Log 功能來印出翻譯結果，並試著透過調整 LINQ 語句來得到更理想的 SQL。

#### 小結
在分析 ORM 的效能影響時，先確定不是誤用造成的效能瓶頸再來討論 ORM 本身對效能的影響會比較恰當。


### 資料庫與 SQL 效能
ORM 能做的就是翻譯，除了翻譯效果不理想的因素外，另外一個面向是從資料庫與 SQL 本身來排查。

#### Cartesian Explosion (笛卡兒爆炸)
這是很經典的問題，網路上資源很多所以這邊就用個範例簡介一下就好。

``` sql
-- Count: 1 AUTHORS * N BOOKS * M AWARDS
SELECT a.NAME AS AUTHOR_NAME, b.TITLE AS BOOK_TITLE, aw.AWARD_NAME AS AWARD_NAME
FROM AUTHORS a
JOIN BOOKS b ON a.AUTHOR_ID = b.AUTHOR_ID
JOIN AWARDS aw ON a.AUTHOR_ID = aw.AUTHOR_ID;
WHERE a.NAME = 'John Smith'
```

假設這個作者關聯 20 本書且他得了 10 個獎項，上面的查詢會查出 200 (10 * 20) 筆資料，依此類推，這個現象在星型架構(Star Schema)的情境會更明顯。但如果我們把它拆成兩個查詢如下，則只會產生總共 30 (10 + 20) 筆資料。
``` sql
-- Count: 1 AUTHORS * N BOOKS
SELECT a.AUTHOR_ID, a.NAME AS AUTHOR_NAME, b.TITLE AS BOOK_TITLE
FROM AUTHORS a
JOIN BOOKS b ON a.AUTHOR_ID = b.AUTHOR_ID
WHERE a.NAME = 'John Smith';
```

``` sql
-- Count: 1 AUTHORS * M AWARDS
SELECT a.AUTHOR_ID, a.NAME AS AUTHOR_NAME, aw.AWARD_NAME AS AWARD_NAME
FROM AUTHORS a
JOIN AWARDS aw ON a.AUTHOR_ID = aw.AUTHOR_ID
WHERE a.NAME = 'John Smith';
```

> 雖然一般情況下，較少的查詢次數能帶來更好的效能，但在笛卡兒爆炸問題下，需要視情況拆分查詢以避免過多資料造成記憶體負擔。大量資料除了對記憶體是負擔外，應用程式處理這些資料也會拖慢執行時間。在極端情況下，多次查詢可能遠快於一次引發笛卡兒爆炸的查詢。

#### 執行計劃 (Execution Plan)
透過執行計劃查看查詢執行的過程，能夠有效的判定出主要效能問題。常見的解決方案如下：

**Index**  
最常見的處理方式，適當的索引能大幅提升查詢速度，而索引種類與細節非常多，需要另外花時間研究。

**Partition**  
索引不是萬能，當一張表的資料量與查詢範圍夠大時，即使使用 Index 進行範圍查詢也會造成效能低落的問題，這時候 Partition (以 Oracle 來說) 就能有效的進一步提升查詢效能。

> 另一個概念是分片(Shard)，[兩者差異參考其他資料](https://medium.com/@_amanarora/partitioning-sharding-choosing-the-right-scaling-method-dbc6b2bec1d5)

**資料歸檔(Archive)**  
將少用的資料 (通常是舊資料) 分散到其他資料庫中，減少常用資料庫中的資料量。但這個策略很依賴商務層面使用資料的方式，只在存在少用資料時才有用。  

另外這個做法很複雜：
1. 首先要先制定資料遷移的時間和相關操作，如果要不停機遷移難度會更高。
2. 查詢時，如果少用資料終究還是要用到，那應用程式中查詢就要考慮兩個資料庫來源，非常麻煩。
3. 承上，跨資料庫查詢對於需要排序與分頁的功能來說是難以跨越的障礙，這個限制會直接限縮這個解法的適用範圍。

#### 小結
資料庫和 SQL 分析範圍很廣，這邊提的是我見過的部分。  

### 總結
會想寫這篇是因為在面對效能問題時，常見到找不到真正的瓶頸但卻花很多精神在微小的改善上的情況，或是沒意識到誤用工具而棄用工具，實在很可惜。  

### 參考
[Partitioning & Sharding — choosing the right scaling method](https://medium.com/@_amanarora/partitioning-sharding-choosing-the-right-scaling-method-dbc6b2bec1d5)  