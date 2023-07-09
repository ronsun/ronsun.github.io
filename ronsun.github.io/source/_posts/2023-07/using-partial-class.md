---
title: C# 部分類別 (Partial Class) 的使用建議
date: 2023-07-09 22:21:32
categories:
- C#
- Language Spec
tags:
---

在 C# 中，partial class 可以讓開發者把一個類別分散到多個檔案中。當程式編譯時，這些分散的部分會被合併成一個單一的類別。這篇會根據以往經驗來紀錄一下使用建議。

<!--more-->

### 設計建議
#### 檔名名稱
檔名名稱應該是 `<Class>.<Category>.cs` 這樣的格式。  

例如一個類別 `Product` 有一個 `Product.cs` 檔案包含了產品的部分資訊，當需要建立另外一個部分類別來包含商品屬性的時候，這個新的部分類別可以放在另外一個 `Product.Properties.cs` 檔案中，這樣做的好處有幾個：
1. `Product.cs` 是主要資訊，其他資訊可以依照分類建立在不同檔案中，非常清楚也容易維護。
2. 使用如果 Visual Studio 開發，建立檔案 `Product.Properties.cs` 的時候它預設的的類別就會是 `Product`，且對程式碼分析工具來說也是符合規範的 (至少我用過的程式碼分析是這樣)。
3. 即便不是使用 Visual Studio 開發，在程式編碼規範的訂定上也比較容易，例如：一般會要求主檔名要符合類別名稱，這在部分類別的情境也適用。  

細節可依狀況調整，但大方向依照 `<Class>.<Category>.cs` 這樣的模式是副作用最小的。

#### 檔案管理
同一個類別的所有部分類別檔案放在同一個資料夾下，避免檔案四散各處而難以追蹤整個類別的完整樣貌。

#### 共用成員的管理
這個主題包含建構子、共用欄位、共用屬性、共用方法等需要在被不同檔案中的部分類別共用的情境。  

通常我會把共用成員放在主要檔案中，例如： `Product.cs` 中就包含所有共用成員，且要維護過程中需要時刻注意呼叫方法時是否會破壞這個原則。

### 適用情境
#### 隔離自動生成 (Auto-Generated)的程式碼
有時候我們需要透過工具或框架自動產生程式碼，最常見的情況就是 ORM (如 Entity Framework）所產生的資料模型 (Entity Data Model)，我們不應該去修改這些自動生成的程式碼，避免之後重新生成覆蓋掉我們的修改，這時候就可以用部分類別來擴充與維護，範例如下：  
``` csharp
// Person.cs
public partial class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime DateOfBirth { get; set; }
}

// Person.Aggregation.cs
public partial class Person
{
    public string FullName
    {
        get => FirstName + " " + LastName;
    }
}
```

這樣一來當重新產生資料模型的時候就只會影響到工具所管轄的檔案 `Product.cs` 而已。

#### 大類別的拆分
有些類別因為定義的過於抽象且現實不允許重構時，就可以透過將一個大類別細分成多個小的部分類別，提升一點可維護性；另外一種是資料容器型的類別 (Model)，尤其是 Context 或是 Metadata 這一類的類別很容易就包山包海，稍微分類一下也是有幫助。  

但是要注意的是：  
**如果適合，一開始就設計成不同類別會更理想；如果不適合設計成不同類別，那再來考慮部分類別。**

#### 分階段重構時
有些包袱特別重的專案，裡面會有很肥又很亂的類別，偏偏又不允許一次到位的重構整個類別，這時候就會需要分階段重構。而分階段重構時必然會面對邊重構邊維護的困境，使得一個原本就很亂的類別中的程式碼新舊交融，因而更難以維護，也容易因為維護不當導致亂上加亂。  

> 這邊的新舊交融不是指重構後留著舊程式碼不刪，而是有些重構了但有些還沒做的意思。

這時候可以把舊的程式碼直接改成部分類別留在原本的檔案 `GhostStory.cs` 中，另外建立一個新檔案 `GhostStory.Refactoring.cs` 包含整理過的程式碼，達到分階段重構期間仍然可以維護舊的程式而不會互相干擾的效果。  

等到幾百年後重構終於完成了，就可以刪掉舊的 `GhostStory.cs` 後再把  `GhostStory.Refactoring.cs` 重新命成回 `GhostStory.cs` 並移除 `partial` 關鍵字就完成了。  

### 缺點與誤用風險
#### 分類不當或程式放在不適合的檔案
用部分類別雖然可以達到大類別分類管理的效果，但如果分類不當或將程式放在不適合的檔案中，反而會增加維護難度。

#### 共用成員難管理
當多個不同檔案的方法需要呼叫共用成員時，那個共用成員要放在哪個檔案也是個不好決定的主題，甚至複雜一點會產生多個檔案間的方法交錯呼叫而難以追蹤與管理。即使一開始決定好了，維護過程也要一直注意呼叫關係是否恰當以及是否要讓成員換檔案住的問題。  

這點是最難掌控的，因為很容易隨著維護的過程慢慢歪掉。

#### 逃避設計或重構
部分類別算是備用方案，如果不先考慮好好設計而一股腦地使用部分類別的話，還是會產生一個大雜燴風格的類別，違反了單一職責原則 (Single Responsibility Principle) 且容易讓程式碼的可維護性快速下降。

### 結論
部分類別是個很好用的工具，但如果使用不當，再好用的工具都會砸到自己的腳，所以設計時應該要綜合考慮優缺點與副作用以及需要的取捨後再做決定。  

### 參考
ChatGPT  

開發經驗  

[Naming Conventions For Partial Class Files](https://stackoverflow.com/questions/1478610/naming-conventions-for-partial-class-files)  

[Best Practices: When not/to use partial classes](https://stackoverflow.com/questions/351272/best-practices-when-not-to-use-partial-classes)