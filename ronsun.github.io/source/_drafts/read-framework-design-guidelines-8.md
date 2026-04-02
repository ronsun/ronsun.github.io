---
title: Framework Design Guidelines 整理與心得 (8)
categories:
  - Reading
  - Framework Design Guidelines
tags:
---

這系列文章的目的是為了之後參考與快速回顧而基於 Framework Design Guidelines 這本書寫的整理與心得，這不是翻譯，所以內容都是閱讀理解後再梳理下來的，和書中語句必然不同，也只會保留我自己覺得需要的部分，並帶一些自己的想法與註解。

本篇包含了第八章的部分。 

<!--more-->

### 陣列 (Arrays)

+ **O 應該** 用集合類型取代陣列。
    > 但效能敏感且長度不變時，還是建議使用陣列。

+ **X 不應** 使用 `readonly` 搭配陣列，因為陣列本身雖然唯獨但內容還是可變的。  
    應該改用其他唯讀型別，例如 `ReadOnlyCollection<T>`。  

+ **O 考慮** 使用不規則陣列 (jagged arrays) 取代多維陣列 (multidimensional arrays)。
    不規則陣列各維度長度可不同，比較省空間，且 CLR 有針對不規則陣列的操作最佳化過，效能會比較好。

BRAD ABRAMS 表示通常 API 都不會使用非一維陣列，除非是本質上就是在解決多維度問題(例如矩陣)。除此之外應該都要使用另外封裝的資料結構或分開的多個陣列。  

### 特性 (Attributes)
+ **O 應該** 以後綴 `Attribute` 命名。

+ **O 應該** 套用 `AttributeUsageAttribute`。
    ``` csharp
    [AttributeUsage(...)]
    public class ObsoleteAttribute{}
    ```
    > 用來限定這個特性能套用到那些範圍，是一種避免誤用的保護措施。

+ **O 應該** 提供含 getter / setter 的非必填屬性，並提供只含 getter 的必填屬性。

+ **X 避免** 在建構子參數中包含非必填屬性。
    換言之，不要讓屬性透過兩個管道 (建構子和 setter) 賦值。   

+ **O 應該** 提供建構子來初始化必填屬性。
+ **X 避免** 提供建構子多載。
    只有一個建構子可以明確區分必填屬性和非必填。  
    > 必填屬性透過唯一的建構子初始化，其他都是非必填屬性，由呼叫端選填。

+ **O 應該** 密封 (Sealed) 客製的特性類別，能讓搜尋時快一點。

### 集合 (Collections)
+ **X 不應** 在公開 API 中使用弱型別結合。
    > 我會進一步避免在所有情境使用弱型別 (非泛型) 集合，主要是難操作以及可能產生的裝箱與拆箱成本。

+ **X 不應** 同時實作 `IEnumerator<T>` 和 `IEnumerable<T>`，非泛型版本亦同。
    換句話說一個型別不能同時是集合與迭代器。  

ANTHONY MOORE 表示設計上，通常要取得最弱的型別作為輸入並輸出最強型別。
> 就是說輸入應該盡量抽象，因為這樣能適應更多不同呼叫端的實作；而輸出要盡量具體，確保呼叫端能把回傳用來做更多事。  
> 但是輸出部分，我認為視情況不一定要用最具體的型別，稍微抽象一點但呼叫端夠用就好，這樣之後內部重構時可以更有彈性而減少變更 API 回傳型別產生破壞性變更的風險。

#### 做為參數

+ **O 應該** 使用最不具體 (最高層基底型別) 的型別做為參數型別，例如 `IEnumerable<T>`。

#### 做為屬性與回傳值

+ **O 應該** 使用 `ReadOnlyCollection<T>` 或其子類別達到真正的唯讀效果。

+ **O 考慮** 實作繼承自 `Collection<T>` 的集合型別而不是直接使用 `Collection`。  
    這樣更好命名且能提供更多的輔助成員達到特定目標。
    ``` csharp
    public TraceSourceCollection: Collection<TraceSource>
    {
        // optional helper method
        public void FlushAll()
        {
            foreach (TraceSource source in this)
            {
                source.Flush();
            }
        }
        // another common helper
        public void AddSource(string sourceName)
        {
            Add(new TraceSource(sourceName));
        }
    }
    ```

+ **O 考慮** 在有唯一辨識元時繼承 `KeyedCollection<TKey, TItem>` 而得到索引帶來的好處。
    但這會更平凡使用記憶體，當這個缺點大於索引帶來的優點時就不應該使用。  

+ **X 不應** 在做為回傳值或是屬性時使其值為 `null`，應該用空集合。


##### 快照與即時集合的比較 (Snapshots Versus Live Collections)
快照指的是某個狀態下的資料，例如讀取資料庫資料後還傳的集合；即時集合內容則是反映即時資訊，例如 `ComboBox`。  

+ **X 不應** 用屬性回傳快照集合，應該回傳即時集合。  
    > 就是屬性回傳內容應該要能反映來源資料 (物件內部的集合) 的變化。

#### 陣列與集合的選用

+ **O 應該** 優先使用集合而不是陣列。

+ **O 考慮** 在非常底層的 API，為了達到更低的記憶體使用足跡，或需要用到執行階段的存取效能最佳化時使用陣列。

+ **O 應該** 使用 `byte[]` 取代位元組集合。

+ **X 不應** 在屬性會複製並回傳一個陣列的情形下使用陣列。
    這是為了避免呼叫端多次在弧圈中多次呼叫該屬性而造成效能問題，如下：
    ``` csharp
    for (int index = 0; index < customer.Orders.Length; index++)
    {
        Console.WriteLine(customer.Orders[i]);
    }
    ```

#### 自定義集合的實作

+ **O 考慮** 繼承自 `Collection<T>`、`ReadOnlyCollection<T>` 或 `KeyedCollection<TKey, TItem>`。

+ **O 應該** 實作 `IEnumerable<T>`。  
    也考慮在合理場景實作 `ICollection<T>` 甚至 `IList<T>`。  

當實作自定義集合時，盡最大可能遵循 `Collection<T>` 和 `ReadOnlyCollection<T>` 實作時的慣例。  

+ **O 考慮** 在經常需要傳給接受非泛型介面 (`IList`、`ICollection`) 的 API 時，同時實作這些非泛型介面。

+ **X 不應** 繼承非泛型型別。
    應該改繼承泛型型別，如：`Collection<T>`、`ReadOnlyCollection<T>` 和 `KeyedCollection<TKey, TItem>`。  

##### 自定義集合的命名

+ **O 應該** 以後綴 `Dictionary` 命名實作 `IDictionary` 或 `IDictionary<TKey, TValue>` 的型別。

+ **O 應該** 以後綴 `Collection` 命名實作 `IEnumerable` 且表示項目清單的型別。  
    ``` csharp
    public class OrderCollection: IEnumerable<Order> {... }
    public class CustomerCollection: ICollection<Customer> {... }
    public class AddressCollection: IList<Address> { ... }
    ```

+ **O 考慮** 以元素型別為前綴 (`AddressCollection` 是 `Address` 這個型別的集合)；若元素為介面，可省略 I (如 `DisposableCollection`)。

+ **O 考慮** 以前綴 `ReadOnly` 命名唯讀集合。  

### `DateTime` 和 `DateTimeOffset`

`DateTimeOffset` 相似於 `DateTime` 但多了一個相較於 GMT 時間的時差。  
> 所以我將他視為含時區資訊的 `DateTime` 來用。

+ **O 應該** 在需要具體時間 (確定的時間點) 時使用 `DateTimeOffset`，即使不知道時區也應使用 UTC。
+ **O 應該** 在時區本身不確定或不存在（如舊資料）時使用 `DateTime`。
> 這兩點看起來矛盾，AI 說明如下：  
> 這兩點看似矛盾，其實是在區分「語意」而不是「是否知道時區」：  
> 若時間本質上是可定位於時間線的時間點（instant），但暫時不知道時區，應以 DateTimeOffset 表示並預設為 UTC（代表「這是一個確定的時間點」）。  
> 若時間來源本身就不包含或無法推斷時區（例如舊資料），則應使用 DateTime（代表「這不是一個可確定的時間點」）。
> 核心原則：不要為不確定的資料補上虛假的時區資訊。  

+ **O 應該** 在需要非具體時間時使用 `DateTime`，例如：開店時間(無論在什麼時區都是早上九點)。

+ **X 不應** 放著 `DateTimeOffset` 不用而用 `DateTimeKind`。
> `DateTimeKind` 僅為標記，不包含完整時區資訊。
> 對 `DateTime` 做時間運算 (如加 8 小時) 不會改變其時區語意，仍維持原本的 Kind。


### `ICloneable`
> 因為該介面設計缺陷與後續發展不符設計初衷，這個介面不應該繼續使用。  
> 取而代之的應該是其他的淺拷貝與深拷貝設計方式。

### `IComparable<T>` 和 `IEquatable<Т>`

`IComparable<T>` 主要用於排序，`IEquatable<Т>` 主要用於查詢。  

+ **O 應該** 在實值型別中實作 `IEquatable<Т>`。  
    `Object.Equals` 可能會因發裝箱，且預設的實作方式因為用到反射而效能不彰。因此，用 `IEquatable<Т>` 能得到更好的效能並避開相關問題。  

+ **O 應該** 在實作 `IEquatable<Т>` 時覆寫 `Object.Equals`。
    ``` csharp
    public struct PositiveInt32 : IEquatable<PositiveInt32>
    {
        public bool Equals(PositiveInt32 other) { ... }
        public override bool Equals(object obj)
        {
            if (!obj is PositiveInt32) return false;
            return Equals((PositiveInt32)obj);
        }
    }
    ```

+ **O 考慮** 在實作 `IEquatable<Т>` 時覆寫 `==` 和 `!=` 運算子。


### 結論

### 參考
ChatGPT