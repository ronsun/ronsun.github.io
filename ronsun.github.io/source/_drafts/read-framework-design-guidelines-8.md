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


#### 快照與即時集合的比較 (Snapshots Versus Live Collections)





### 結論

### 參考
