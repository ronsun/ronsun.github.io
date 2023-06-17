---
title: 使用 ORM 做單次大查詢的效能陷阱
date: 2023-06-17 23:44:28
categories:
- Others
tags:
---

在開發過程中，我們經常需要透過 ORM　(我是使用 EF Core) 從資料庫中取得資訊，在查詢很複雜的時候我通常都會盡量透過一個查詢就把資料準備好，並轉換成巢狀的資料結構以便於應用程式進行操作，這樣可以避免多次查詢造成的效能風險。　　

但前陣子在開發過程檢查 ORM 產生的 SQL 時意外發現我的認知不一定是對的。  

<!--more-->

### 問題
因為單次大查詢雖然能避免多次來回查詢的成本，但是當遇到巢狀的資料結構的時候，單次大查詢轉換出來的 SQL 可能會非常糟，問題主要可以分成兩個方向。

這邊先附上一個複雜的 LINQ 查詢與透過 EF Core 轉化出來的 SQL 作為下面說明的參考資料：

``` csharp
var courseDetails = _dbContext.Set<Course>()
    .Where(c => c.Id == id)
    .Select(c => new Detail
    {
        Id = c.Id,
        Name = c.Name,
        Teacher = c.Teacher.Name,
        Students = c.Students.Select(s => new Student
        {
            Id = s.Id,
            Name = s.Name,
            Grade = s.Grades.FirstOrDefault(g => g.CourseId == c.Id).Grade,
            PersonalInfo = new PersonalInfo
            {
                Address = s.PersonalInfo.Address,
                PhoneNumber = s.PersonalInfo.PhoneNumber,
            }
        }).ToList(),
    })
    .SingleOrDefault();
```

``` sql
SELECT 
    "c"."Id", 
    "c"."Name", 
    "t"."Name", 
    "s"."Id", 
    "s"."Name", 
    (SELECT "g"."Grade" FROM "Grades" AS "g" WHERE ("g"."CourseId" = "c"."Id") AND ("g"."StudentId" = "s"."Id") FETCH FIRST ROW ONLY),
    "p"."Address", 
    "p"."PhoneNumber"
FROM 
    "Courses" AS "c" 
    LEFT JOIN "Teachers" AS "t" ON "c"."Id" = "t"."CourseId"
    JOIN (
        SELECT "s".*, "cs"."CourseId" 
        FROM "Students" AS "s"
        INNER JOIN "CourseStudents" AS "cs" ON "s"."Id" = "cs"."StudentId"
        ORDER BY "cs"."CourseId"
    ) AS "s" ON "c"."Id" = "s"."CourseId"
    LEFT JOIN "PersonalInfos" AS "p" ON "s"."Id" = "p"."StudentId"
WHERE 
    "c"."Id" = :id
ORDER BY 
    "c"."Id", 
    "s"."CourseId"
```

> 這個問題是工作上遇到的也**已經確認過問題的確存在**，但當然不能拿公司實際資料當範例，所以這邊用 ChatGPT 產生 LINQ 和 SQL 來當範例，因為是機器產生的和實際運作產生的 SQL 不一定會相同甚至無法運作，**但作為展示概念的參考資料已經足夠了**。

#### 大量多餘的資料
這部分可以用 SQL 搜尋出來的資料是一個表格為出發點去理解，因為透過 SQL 沒辦法直接搜尋出一個巢狀的資料結構，所以在查到的資料到應用程式的巢狀類別中間勢必要經過轉化，而這個轉化是發生在應用程式中的。  

從上面的範例來看，我們可以看到他產生的 SQL 因為包含大量的 JOIN 所以搜尋出來的資料筆數可能會極大，然後才在應用程式中分組整合。

**而重點在於這大量的資料中的多數欄位都是重複的內容。**  

> 以實際遇到的情境來說，我只是要搜尋一筆主資料，但因為關聯到很多其他資料表所以實際上搜尋出幾千筆的資料，然後在應用程式中將幾千筆資料分組整合成一個巢狀類別。  

這會造成的問題有：  
+ **無謂的記憶體消耗，甚至 GC 的負擔**  
透過上面描述，可想而知這幾千筆中重複的內容會造成多少無謂的記憶體消耗，如果資料夠大還可能會進一步造成 GC 的負擔。
+ **提高對 CPU 的負擔**  
這點也很好理解，大量資料的轉化需要很多運算資源。

#### 不必要的排序
另外一個就是不必要的排序，從前面的範例可以發現所產生的不必要的排序，這個排序會使得 SQL 本身的查詢成本大幅提高。  

> 至於為什麼會排序，根據 ChatGPT 的說法是：  
> Entity Framework Core 在產生 SQL 語句時，會在某些情況下插入排序操作，以確保返回的資料符合 C# 的 LINQ 查詢的順序。這通常發生在涉及集合導航屬性的查詢中，因為 EF Core 需要保證資料的順序與原始查詢相符。  
> 這種排序操作是 EF Core 為了處理導航集合屬性（如 c.Students）而添加的。它需要按照外鍵（在這裡是 CourseId）的順序來獲取相關的 Students，以便能夠正確地建立回傳的物件。

### 解決方案
解決方式其實就是把單次大查詢轉化成多次小查詢，如下範例。

``` csharp
var courseDetails = _dbContext.Set<Course>()
    .Where(c => c.Id == id)
    .Select(c => new Detail
    {
        Id = c.Id,
        Name = c.Name,
        Teacher = c.Teacher.Name,
    })
    .SingleOrDefault();

courseDetails.Students = _dbContext.Set<Student>()
    .Where(s => s.Courses.Any(c => c.Id == courseDetails.Id))
    .Select(s => new Student
    {
        Id = s.Id,
        Name = s.Name,
        Grade = s.Grades.FirstOrDefault(g => g.CourseId == courseDetails.Id).Grade,
        PersonalInfo = new PersonalInfo
        {
            Address = s.PersonalInfo.Address,
            PhoneNumber = s.PersonalInfo.PhoneNumber,
        }
    }).ToList();
``` 

因為被拆成幾段小查詢，所以每一段查詢都會降低 JOIN 的數量 (甚至沒有)，因此就能避免單次大查詢的效能陷阱了。

> 但這邊還是要注意，不是把巢狀拆掉分開就是好的，如果 JOIN 後的資料量很小，多次小查詢的效能是可能比單次查詢要差的，這部分就變成要因地制宜來判定了。

### 總結
以前總是覺得多個小查詢查資料效能比較差，沒想到其實不一定。  

總結一下巢狀物件的查詢心得：  
1. 看到巢狀物件考慮拆成多個小查詢。
2. 巢狀部分的資料量很大或預期會快速成長時，拆成小查詢可能比較好。
3. 巢狀部分的資料量很小且幾乎不會增長時，不拆可能程式碼會比較整潔。
4. 拆與不拆之間，要綜合考量可讀性、效能差異幅度、資料是否會增長...等多個要素，也可將 ORM 所產生的 SQL 印出來後分析以協助判斷。

### 參考
ChatGPT