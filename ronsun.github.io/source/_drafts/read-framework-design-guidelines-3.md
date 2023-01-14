---
title: Framework Design Guidelines 整理與心得 (3)
categories:
  - Reading
  - Framework Design Guidelines
date: 2022-12-27 23:15:09
tags:
---

這系列文章的目的是為了之後參考與快速回顧而基於 Framework Design Guidelines 這本書寫的整理與心得，這不是翻譯，所以內容都是閱讀理解後再梳理下來的，和書中語句必然不同，也只會保留我自已覺得需要的部分，並帶一些自己的想法與註解。

本篇包含了第三章的部分。 

<!--more-->

> 命名的重要性常常被忽略，甚至被認為是鑽牛角尖的主題，但它其實非常的重要，甚至這本書用了一整章的篇幅來說明。
> 命名的重要性大概有幾個方向:  
> 1. **一致的大小寫規則** 可以讓使用一看就知道這個成員代表的是什麼，例如: 欄位(Field)、屬性(Property)、常數(Constant)、區域變數、參數等等。
> 1. **一致的命名風格(包含大小寫、前後綴、常用詞、用詞規範...)** 可以讓使用者非常快速的知道用途，甚至透過規則去"猜"到類似的功能而間接提高使用者體驗。

## 大小寫的慣例

### 基本大小寫規則
具體分兩種，雙駝峰 (PascalCasing) 與單駝峰 (camelCasing)，並且不要使用蛇形命名法 (snake_case)。


> 單駝峰或雙駝峰的適用情境如果要特別記太麻煩了，一般情境下只需要知道  
> 
> **單駝峰適用於**  
> + 參數 (Parameter)
> + 區域變數 (Variable)
> 
> **特殊情境**
> + 私有實例欄位 (Private Instance Fields) 用底線加單駝峰，例如: `private int _privateField;`
>     - 實例欄位 (Instance Fields) 必須是私有的，如果需要開放要改用屬性 (Property)。
>     - 這邊所提的規則不包含靜態欄位。
> 
> **雙駝峰適用於**
> + 除以上情境以外都用雙駝峰

### 縮略字 (Acronyms) 的大小寫規則
一般來說要避免使用縮略字，除非是通用用法，例如: Html、Xml、IO 等。
而縮寫 (Abbreviation) 是另外一個概念，不應該使用，例如: Amt 是 Amount 的縮寫。

> 不應該縮寫的原因主要是一旦用了縮寫，程式碼中就容易出現一個單字有完整版與縮寫版兩種，而且縮寫在口語上很難表達，容易造成溝通障礙或誤解，而通用縮略字可以接受是因為他已被廣泛接受，且縮略字的全文通常很長.

縮略字 (Acronyms) 的大小寫規範主要分兩種情境:  
+ 兩個字元的縮略字，要不是全大寫就是全小寫，以 `IO` 為例如下:
  - `System.IO`
  - `public void StartIO(Stream ioStream)`
+ 其餘縮略字 (Acronyms) 同一般情境，開頭大寫其餘小寫。

### 合成字 (Compound Words) 與常用詞的大小寫
合成字 (Compound Words) 如果可以作為單一個單字，則應視為一個單字，例如: Endpoint 視為一個字而不是 End Point，為了區分能不能視為一個單字，可以看最新的字典中是否這樣用 (Written in closed form)。

下表列了一些合成字與常用字的大小寫和拼字規則:  
+ **B**it**F**lag
+ **C**allback
+ Canceled (不使用兩個 L 的 Cancelled)
+ **D**o**N**ot (不使用 Don't)
+ **E**mail
+ **E**ndpoint
+ **F**ile**N**ame
+ **G**ridline
+ **H**ashtable
+ **I**d
+ Indexes (不使用 Indices)
+ **L**og**O**ff (不使用 LogOut)
+ **L**og**O**n (不使用 LogIn)
+ **S**ign**I**n (不使用 SignOn)
+ **S**ign**O**ut (不使用 SignOff)
+ **M**etadata (單駝峰時全部小寫 metadata)
+ **O**k
+ **P**i
+ **P**laceholder
+ **U**ser**N**ame
+ **W**hite**S**pace
+ **W**ritable

> 常用字用法真的非常非常瑣碎，真的要規定的話一定要明確列表出來，且必須用靜態程式碼工具 (需要自訂規則) 來抓不一致的部分，這種東西用人工審查來確保非常沒效率且不可靠。

### 大小寫敏感
+ **X 不應** 用相同的名字但不同大小來區分變數，就算語言支援也一樣。
    > 我曾經以為不會有人這樣做，但後來還是在專案中看到了。

## 通用命名慣例
### 選詞
命名應該能充分表達目的與功能，所以名字簡短與精確就顯得更為重要，名字應該要能表達情境並且要是所有人都理解的概念而不是技術用語。

> 就是命名應該表達 "這在做什麼" (例如: Contains) 而不是 "這是怎麼做的" (例如: KMPMatching)。

+ **O 應該** 選擇易讀通順的名字，以英文來說 `HorizontalAlignment` 比 `AlignmentHorizontal` 通順

+ **O 應該** 優先使用易讀而不是簡潔的名字 (前提是兩者不能兼具)，例如: CanScrollHorizontally 優於 ScrollableX 這種模糊的名字。
    > 另外就是詞性與單複數問題，尤其是詞性不對在後續維護時很容易間接導致命名不一致，例如: Transaction**Approval** 和 Adjustment**Approve**。  
    > 這種不一致雖然不致命，但如果能注意是更好，其他還有拼字、同義詞、相似詞的選用等細節也是要注意的。

+ **X 不應** 使用底線、減號或是英文和數字以外的字元來命名。
    > 如果用全中文命名的話，那就是依此類推。

+ **X 不應** 使用匈牙利命名法。
    > 介面用 `I` 開頭是一個特例。
    >
    > 合理的前後綴不屬於匈牙利命名法的範圍，當難以分辨是否是合理的前後綴時，我會換個角度從使用前後綴的命名方式會帶來的缺點思考，只要沒有這些缺點就視為合理的前後綴。
    >
    > 匈牙利命名法或類似風格的命名的缺點有:
    > 1. **干擾閱讀**，例如: iAge 會先被那個表示 int 的 `i` 干擾一下。
    > 1. **冗餘且沒幫助**，例如: `int iAge = 18;`，看型別就知道是 int 了，不應該加前綴 `i` 來標示。
    > 1. **隨著開發會常改名**，如果忘記改名會變成誤導後人的地雷，例如: `int iAge = 18;`，變數要改成 `uint` 型別時就連變數名都要改，便成 `uinit uiAge = 18`，萬一忘了改就變成會騙人的程式碼了。
    > 如果這個例子不夠強烈，再舉些工作中常見的例子如下: 
    >    * `FindDataOver3Months()` 哪天需求改成 6 個月通常改的人不會記得 (或是不願意) 改名字，最後變成名字叫 `FindDataOver3Months()` 但內容是查詢過期 6 個月的資料，這種類型的 API 叫 `FindExpiredData()` 加上一點註解說明就會好很多。
    >    * `FindById(int id)` 這種 FindByXXXAndYYY 的命名模式也是常見的問題，哪天加個參數 (尤其是選擇性參數)，方法名字馬上變成錯的了，這種的改成 `Find(Condition condition)` 就會精確很多，且增刪改條件完全不會影響方法簽章。
    >
    > 總之，前後綴可以用，但要能合理、穩定且有有效再用。

+ **X 避免** 用會和語言關鍵字衝突的命名，雖然 C# 有提供 `@` 來跳脫，例如: `object @class = null;` 是合法的，但總是會增加沒必要的干擾，也沒什麼明顯的好處。
    > 更別用 `object clazz = null;` 這種刻意拼錯的奇特做法來規避。

### 縮寫 (Abbreviation) 和 縮略字 (Acronyms) 的使用
一般來說不要用，除非是多數人都這樣用的慣用字。

+ **X 不應** 使用縮寫或是縮略字來命名
    > 除非多數人都這樣用，且原本的全名不會被使用，否則一來後人看不懂，二來會使得程式碼中對同一個命名出現兩種拼字方式，像是有時用 `adj` 但有時又用 `adjustment` 這種事不要做。

+ **X 不應** 使用未被廣泛接受的縮略字。

### 避免命名中包含語言規格
因為框架設計是給多語言使用的，所以應該避免用語言規格命名，用 JEFFERY RICHTER 舉的例子來說，在 VB 中是可以拋出 `NullReferenceException` 的但 `null` 是在 C# 中的語言規格，在 VB 中應該是 `Nothing`.

+ **O 應該** 用語義化的名字取代語言中的關鍵字，例如: `GetLength` 就優於 `GetInt`。

+ **O 應該** 使用 CLR 的型別取代特定語言的規格作為名字，例如: 應該使用 `ToInt64` 而不是 `ToLong`。
    > JEFFERY RICHTER 也認為應該禁止使用特定語言中的別名 (例如: C# 中的 `short`、`int`、`long` 其實是 CLR 型別 `Int16`、`Int32`、`Int32` 的別名) 作為命名    

+ **O 應該** 在**很偶爾且很難用有意義的命名時**，用 `item` 或 `value` 這種抽象的命名來取代用特定型別名稱來命名，例如: `void Write(double value)`、`void Write(float value)`、`void Write(short value)`。
    > 另外一種情竟是變數生命週期短的時候，例如 Lambda 表示式中的引數，或是短暫需要的變數。 但還是要謹慎考慮在適當的時候使用，不是想不到名字就可以這樣用。

> 小結一下:  
> 這一節的核心在於，如果框架設計出來有需要支援多語言，那就不要使用特定語言的規格作為命名避免跨語言後造成混淆。

### 新版本 API 的命名
通常面對新功能的時候我們會優先考慮擴充現有的程式，但有時候現況不允許擴充的時候，就變成要建立一個新的類別或成員，這種情境下不同版本的命名就很麻煩，這一節提供一些建議。

+ **O 應該** 把新版 API 命名成和舊版相似的名字，便於暗示這兩個 API 是前後版本的關係。
**O 考慮** 使用全新且語意清晰的命名作為新的 API 的名字，用前後綴是不得已的做法。
**O 應該** 在 "不得已" 的時候優先使用後綴來標示新版 API，這個原因是在排序和 IDE 自動提示上，他們兩者通常會排在緊鄰的位置上，方便搜尋、閱讀與理解，例如 `ReaderWriterLockSlim` 是效能強化版 (但不向下相容) 的 `ReaderWriterLock`。
**O 應該** 在 "萬不得已" 的時候 (找不到任何合理的命名與後綴時)，在舊版 API 名字後面後綴一個數字做為新版 API 的命名，例如:  `X509Certificate` 的新版 API 是 `X509Certificate2`。  
而不管最終怎麼做，都應該在舊版 API 上加上 `[Obsolete("Reason...")]` 來標示舊版的棄用並說明替代方案。
    > 走到後綴數字已經很糟了，如上所說是萬不得已的作法。

KRZYSZTOF CWALINA 表示，他們曾經試著推出 `TimeZone2` 這個型別並成為社群的爭議點，最後才改名成沒那麼糟的 `TimeZoneInfo`，神奇的是同樣情境的 `X509Certificate2` 卻沒受到反彈，他推測是人們對於少用的功能會比較能接受命名瑕疵。

+ **X 不應** 使用 "Ex"、"New" 或類似的後綴來命名新版 API。
    > 書中雖然沒說為什麼，但不難理解，如果 `Foo` 的下一版叫做 `FooNew`，那再下一版又遇到同樣的問題怎麼辦？ 
    > 這也不是都市傳說，我曾經看過一個 API 叫做 `NewNewSomething`，當時真的是驚呆了。

## 組件 (Assemblies) 與動態連結程式庫 (DLLs) 的命名
組件和 DLL 幾乎都是一對一的關係，所以這裡的命名方針雖然是針對 DLL 的，但同時也對應到他唯一的組件命名。

+ **O 應該** 選擇能包含大量功能的 DLL 名字 (儘量抽象廣泛)，例如: System.Data.dll。

+ **O 考慮** 用 `<Company>.<Component>.dll` 這個模式命名 DLL。 Component 的部分可以用 `.` 區隔成多個部分，例如: `Microsoft.VisualBasic.Vsa.dll` (Component 部分為 `VisualBasic.Vsa`) 是合理的。

## 命名空間 (Namespaces) 的命名
針對命名空間推薦的命名規則如下:  
`<Company>.(<Product>|<Technology>)[.<Feature>][.SubNamespace]`   
> 就是依範圍、分類由大到小依序排列，中間用 `.` 隔開。

+ **O 應該** 以公司名稱開頭，避免不同公司相同的命名空間讓使用者同時使用時出問題。
    BRAD ABRAMS 表示用公司 "全稱" 命名很重要，不然使用者不一定會知道縮寫代表的是什麼公司。

+ **O 應該** 使用穩定且不會因為版本更新而改變的產品做為第二層的命名。
    BRAD ABRAMS 表示別用營運端想的花俏名字當第二層產品層的命名，基於營運目的所用的名稱可能會隨時間與版本改變，但是命名空間中產品層會的部分影響該產品所有程式碼的命名，所以選擇技術上合理 (Technically Sound) 的名字。

> 這點在專案設計上也很有用，很多人在命名的時候都會第一時間想拿 "頁面顯示" 的用詞來命名，但一方面這些名詞有時候是為了顯示而定的，從技術面來看名字可能不恰當，另一方面需求是會改變的，長期維護下來展示層的名字必然經常更動，所以在設計時的命名不必然要和展示層的命名對齊。
> 也不是說命名一定要不一樣，事實上能一樣是最好，但應該以對系統而言穩定合理的名字為優先。

+ **X 不應** 使用使用組織架構的名字來當命名空間的名字，因為組織架構是隨時會因為改組而改變的，應該用相關的技術名詞來命名。
    > 例如用部門名字當命名，當部門改組的時候就會很麻煩。

+ **O 應該** 使用雙駝峰並用 `.` 來區分層級，例如: `Microsoft.Office.PowerPoint`，但如果品牌名稱的大小寫和規範不同，應該以品牌名稱的大小寫優先.

+ **O 考慮** 使用複數來給命名空間取名，例如: `System.Collections` 優於 `System.Collection`. 但是品牌名稱或是縮略字是例外，例如 `System.IO` 就優於 `System.IOs`。

+ **X 不應** 讓命名空間和其下的型別的名字重複，很多編譯器不支援。

> 總之，基本上命名就是依照規範，但遇到和產品或品牌名字衝突時以產品優先，反正就是產品最大。

### 關於命名空間和型別名稱的衝突
命名空間用於將型別收納於成一個易於理解的架構，但是不同命名空間下的相同型別會造成混淆而導致開發者需要額外費心去處理這個問題。

+ **X 不應** 使用太過於泛用的名稱，因為很容易跟其他命名空間下的型別撞名，例如: `Element`、`Node`、`Log`、`Message` 應該改用更精準的名字如 `FormElement`、`XmlNode`、`EventLog`、`SoapMessage`。
    > 從另外一個面向，也適用於要避免自己的框架或套件的型別和別的的套件 (會一起使用) 中的型別撞名時使用。

> 依照 `Application Model 的命名空間`、`基礎建設 (Infrastructure) 的命名空間`、`核心 (Core) 功能的命名空間`、`技術相關 (Technology) 的命名空間` 分類下，還有一些細節，但是內容幾乎都一樣，就是不要在相同群組的命名空間下提供重複的類別名稱，如果有適當的用命名空間分類的話，這些問題應該不會發生。


## 類別 (Classes)、結構 (Structs) 、與介面 (Interfaces) 的命名  

+ **O 應該** 使用名詞或名詞片語來命名類別和結構，因為他們表達的是系統中的 "某個事物"。
  不同於此，方法 (Methods) 是以 "動詞片語" 來命名。

+ **O 應該** 使用形容詞或形容詞片語來命名介面，因為介面代表的是 "某種能力" 例如: `IComparable<T>` 表示有比對的能力；`IForamttable` 表示有格式化成字串的能力；`IDisposable` 表示有釋放資源的能力。
    > 介面命名有另外一種替代方案是 `ICan*****`，同樣是表達有某種能力，但終究是替代方案，應該留到真的想不到名字的時候才用。

而如果無法如上述這樣命名，那就要反過來想想型別的設計是否不妥了。

另外還有一點重要的是，簡短好用的名字要留給最常用的型別，即使有其他更適用但很不常用的型別，也應該以常用優先，這是基於使用者友善考量。
> 不過這是在命名對雙方來說都很合理的前提下，不是說要硬套。

+ **O 考慮** 使用基底類別的名字當衍生類別名字的後綴。
    這能有效的描述繼承關係，例如: `Exception` 的所有衍生類別都是以 "Exception" 為後綴，`Stream` 的衍生類別亦如此.
    > 算是合理的後綴

+ **O 應該** 在介面的名字前面加前綴 `I` 來表示它是個介面。
    > 匈牙利命名法的特例

+ **O 應該** 確保介面 `IFoo` 的預設實作的名字是 `Foo`，只差前綴的 `I`。

### 泛型型別參數的命名
+ **O 應該** 使用有意義的型別參數，除非單一字母有足夠的表達能力且其它命名不會更好。

+ **O 應該** 在型別參數前面加上前綴 `T`，例如: `TSession`。

+ **O 考慮** 在適合使用單一字母作為型別參數的情境下使用 `T` 做為型別參數。

+ **O 考慮** 加上型別參數的條件約束，例如加上型別參數 `TSession` 的條件約束限制它必須是 `ISession` 或其衍生型別，如下範例:  
    ``` csharp
    public interface ISessionChannel<TSession>
        where TSession : ISession
    {
        TSession Session { get; }
    }
    ```
    
### 一般型別的命名
如果型別是衍生自 .NET Framework 提供的型別，要依照下表的規則命名，且後綴的規則是要套用到所有衍生類別 (包含更下層的所有衍生類別)。另外這些後綴是保留給衍生類別使用以利於辨識的，不要用在其他不相關的地方，例如 `public class WindowsAttribute : Control {}` 是不正確的。

> 下表有點繁瑣，不想記這些東西可以回到基本觀念 — 從 .NET Framework 中找範例並使用相同的規則。
> 而從 [.NET Framework 官方文件 (System.Attribute)](https://learn.microsoft.com/en-us/dotnet/api/system.attribute?view=netframework-4.8.1) 可以看到各種衍生類別，通常能找出命名規則。
> (說是這樣說，但官方有時候也會出現規則外的特例就是。)

| 基底類別 |  衍生類別規範 | 
|---------|---------------|
|  `System.Attribute`  | **O 應該** 加上後綴 `Attribute` |
|  `System.Delegate`   | **O 應該** 在事件的名字加上後綴 `EventHandler`。 |
|                      | **O 應該** 在 Event Handler 以外的名字使用後綴 `Callback`. |  
|                      | **X 不應** 使用後綴 `Delegate`。 |
|  `System.EventArgs`  | **O 應該** 使用後綴 `EventArgs`。 |
|  `System.Enum`       | **X 不應** 繼承這個類別，改用語言支援的關鍵自，例如 C# 中的 `enum`。 |
|                      | **X 不應** 使用後綴 `Enum` 或 `Flag`。 |
|  `System.Exception`  | **O 應該** 使用後綴 `Exception`。 |
|  `System.IO.Stream`  | **O 應該** 使用後綴 `Stream`。 |
|  `IDictionary`、`IDictionary<TKey，TValue>`  | **O 應該** 使用後綴 `Dictionary`，要注意的是 `IDictionary` 是一種特殊的集合 (Collection)，這邊的規則優先於一般集合的規則。 |
|  `IEnumerable`、`ICollection`、`IList`、`IEnumerable<T>`、`ICollection<T>`、`IList<T>` | **O 應該** 使用後綴 `Collection`。 | 
|  `CodeAccessPermission`、`IPermission` |  **O 應該** 使用後綴 `Permission`。 |

### 列舉 (Enumerations / Enums) 的命名
除大小寫外還有其他規則。

**O 應該** 使用單數，但位元欄位 (Bit Fields) 應該使用複數。
``` csharp
public enum ConsoleColor
{
    Black,
    Blue,
    Cyan,
    ...
}
```

``` csharp
[Flags]
public enum ConsoleModifiers
{
    Alt,
    Control,
    Shift,
}
```

> 概念上就是: 列舉名稱是一種"種類"，所以是單數；但位元欄位是多選項的組合則是複數。

**X 不應** 使用後綴 `Enum` 或 `Flag`，也不應該在列舉項目中使用 Enum 的名稱當前綴，例如 `ImageModeEnum.ImageModeEnumBitmap` 應該改成 `ImageMode.Bitmap` 才對。


### 型別成員的命名

#### 方法 (Methods) 的命名  
**O 應該** 用動詞或動詞片語來命名，因為方法是用來 "執行一個動作" 的。

STEVEN CLARKE 表示要盡量用功能來命名而不是實做細節。
> 這個問題最常見的是 `GetOrdersPendingOver3Days()` 這種命名，應該改成 `GetExpiredOrders()`，而不是把"逾期"的條件一個一個串到名字上。 
> 也就是說 "找出逾期的訂單" 是功能，而 "找出等待超過三天的訂單" 是實作細節。

#### 屬性 (Properties) 的命名  
**O 應該** 用名詞、名詞片語或是形容詞來命名屬性。

**X 不應** 讓屬性名 `Foo` 和方法 `GetFoo()` 同時存在。
> 在 C# 中，屬性背後是欄位和方法 (getter / setter) 的組合，所以上例的 `GetFoo()` 會造成很大的混淆。

**O 應該** 用複數單字來表達集合型別的屬性，而不是使用後綴，例如: `public List<string> NameList { get; }` 是不好的，應該改成 `public List<string> Names { get; }`。

**O 應該** 用肯定句 (Affirmative Phrases) 來命名 `Boolean` 型別的屬性。 如果必要也可以前綴 `Is`、`Can` 或 `Has` (但一般來說前綴都是多餘的)。
> 有一派習慣是讓布林值的預設永遠是 `false`，好處是看到布林值時不需要去想預設是什麼，但這時候命名會比較麻煩，例如: `Allowed` 代表是否允許，但當預設值要是允許的時候就會出現不能用 `NotAllowd` (預設 `false`) 來命名但又需希望預設永遠是 `false` 的兩難。
> 這時候可以考慮使用反義詞，例如這個例子命名可以考慮改用 `Blocked` 代表是否阻擋，這樣就順理成章的變成了 `Blocked`(預設 `false`)。
> 但要注意的是，如果用反義詞會造成維護上或溝通上的困難時，應該回過來放棄布林值預設 `false` 的規則。

**O 考慮** 讓屬性用和它型別一樣的名字，例如: `public Color Color { get;set; }`。

#### 事件 (Events) 的命名
**O 應該** 用動詞或動詞片語來命名， 因為事件表示的是"一個動作"的發生，例如: `Cliked`、`Painting`。

**O 應該** 利用時態讓事件的名字有執行前或執行後的概念，分別用現在式或是過去式。
例如一個表示 `關閉` 的事件，在頁面關閉前觸發的事件叫做 `Closing`，在事件關閉後觸發則叫做 `Closed`。

**X 不應** 用 `Before` 或 `After` 當作前後綴來命名，應該如上述用時態表示。

**O 應該** 用 `EventHandler` 當後綴來命名 Event Handler，例如:  
`public delegate void ClickedEventHandler(object sender，ClickedEventArgs e)`。

**O 應該** 使用固定的命名 `sender` 和 `e` 來命名參數，`sender` 表示發起這個事件的物件，並且不管實際型別是什麼都應該固定用 `object` 型別。

**O 應該** 用 `EventArgs` 為後綴來表示事件參數所用的類別。

#### 欄位 (Fields) 的命名
這邊提到的大小寫與前後綴規則都只包含 `staic public` 或 `static protected` 欄位。
> 這邊我是理解成不包含私有實例欄位 (Private Instance Fields)。

**O 應該** 用**名詞**、**名詞片語**或是**形容詞**來命名欄位。

**X 不應** 用前後綴，例如: `g_` 或 `s_`。


#### 參數 (Parameters) 的命名

**運算子多載參數 (Operator Overload Parameters) 的命名**  
**O 應該** 在沒有更有意義的命名時，使用 `left` 和 `right` 作為位元運算子 (Binary Operator) 多載參數的命名，例如:  
`public static Timspan operator -(DateTimeOffset left，DateTimeOffset right)`。

**O 應該** 在沒有更有意義的命名時，使用 `value` 做為一元運算子 (Unary Operator) 多載參數的名字，例如:  
`public static BigInteger operator -(BigInteger value)`。

**X 不應** 使用縮寫或數字來作為運算子多載參數的命名，例如:  
``` csharp
// 錯誤的命名
public static bool operator ==(DateTimeOffset d1，DateTimeOffset d2)
```

#### 語系檔 (Resources) 的命名
**O 應該** 在高可讀性的前提下追求簡潔，不要為了省空間犧牲可讀性。

**O 應該** 用雙駝峰命名語系檔的 Key。
**O 應該** 只用英文字母和底線命名。
> 把語系檔的 Key 視為程式碼的一部分來命名，且我會希望連底線都不要用。

**O 應該** 用 `例外型別的名字 + 簡短的名字` 來命名例外訊息 (Exception Message) 所需的語系檔的 Key，例如:  
``` csharp
// Illegal chracters for ArgumentException
ArgumentExceptionIllegalCharacters
// Invalid name for ArgumentException
ArgumentExceptionInvalidName
// Message about malformed file name for ArgumentException
ArgumentExceptionFileNameMalformed
```

### 結論
遵循這樣的規則可以讓程式碼更一致，對使用者來說也比較易用。

> 總之就是幾點  
> 1. 命名清晰，簡潔很好但易讀優先. 
> 1. 大小寫一致，前後綴合理。
> 1. 用固定的模式命名，最好能讓使用者透過聯想就猜測到其他相關功能。