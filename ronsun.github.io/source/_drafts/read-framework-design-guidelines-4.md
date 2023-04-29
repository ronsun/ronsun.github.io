---
title: Framework Design Guidelines 整理與心得 (4)
categories:
  - Reading
  - Framework Design Guidelines
date: 2023-07-08 23:57:20
tags:
---

這系列文章的目的是為了之後參考與快速回顧而基於 Framework Design Guidelines 這本書寫的整理與心得，這不是翻譯，所以內容都是閱讀理解後再梳理下來的，和書中語句必然不同，也只會保留我自已覺得需要的部分，並帶一些自己的想法與註解。

本篇包含了第四章的部分。 

<!--more-->

從 CLR 的角度，型別只有參考型別和值型別兩種，但針對框架設計可將型別分成幾組如下：
+ 參考型別 (Reference Types)
  - 類別 (Class)：參考型別中常見的形式。
  - 靜態類別 (Static Classes)：用於收納靜態方法，通常用於其他功能的輔助方法 (shortcuts to other operations)。
  - 集合 (Collections)
  - 陣列 (Arrays)
  - 例外 (Exceptions)
  - 特性 (Attribute)
+ 值型別 (Value Types)
  -  結構 (Structs)：值型別中常見的形式，應該保留給像是語言原生型別(Primitive Tyeps) 這種小型簡單的型別。
  -  列舉 (Enums)  
     值型別中的特例，用於定義像是顏色等"特定的種類"。
+ 介面 (Interfaces)

---

+ **O 應該** 確保型別中的成員符合型別的定義，而不是隨便把幾個方法隨便收成一個類別，類別應該也要能用簡單的一句話就能描述，且應該能精準定義才能讓他能只收納與定義精準符合的方法。
  > 把類別定義得過於抽象或職責包山包海導致類別無限膨脹是最常見的，嚴重的還有一些方法被放進與他不相關 (或僅是間接相關) 的類別中造成維護上很大的混淆與困擾。  
  > 另外，有些人會覺得類別多難管理，但通常是因為類別沒定義好、命名空間或資料夾結構沒規劃好等等的因素導致的，主因不在類別的數量。

JEFFRY RICHTER 表示有些類別最後會變成一堆低相關性的方法的集中地，像是 .NET Framework 提供的 Marshal，GC，Console，Math，Application 等。

> **總之，類別應該精準、單一職責且盡量小巧。**

## 型別和命名空間
在框架開發前就應該要決定好命名空間與其職責，這種由上而下 (top-down) 的設計之所以重要是因為他能確保不同命名空間之間的型別能順利的協作且不會發生衝突、矛盾與重複。

+ **O 應該** 將命名空間依功能區分成不同的階層，階層設計應該根據使用者的需求與瀏覽方便性來做最佳化。

KRZYSZTOF CWALINA 表示命名空間不應該是用來避免型別名稱重複的機制，同一個框架出現同名型別在不同的命名空間是草率的決定。  

> 仔細想想以往應用程式的設計上經常犯這個錯誤，實務上的確也引起了不小的混淆與使用上的困擾，更麻煩的是者可能是因為早期命名空間規劃不當而間接引起的，要處理還得改動命名空間的設計。
 
+ **X 避免** 過多層的命名空間，會造成使用者因為需要經常切換階層而難以瀏覽。

+ **X 避免** 規劃太多命名空間，一般情況不該讓使用者引入 (import) 太多命名空間，常用型別應該盡可能收納在同一個命名空間中。
    > 不過還是要合理，不應該超出該命名空間定義的範圍。

JEFFRY RICHTER 表示有一個範例：`System.SerializableAttribute` 和 `System.NonSerializedAttribute` 因為收納在不洽當的命名空間下，使得使用者沒意識到他們應該是和 `System.Runtime.Serialization` 命名空間下的型別搭類使用的，而經常誤將他們和 `System.Xml.Serialization` 下的 `XmlSerializer` 搭配使用，實際上 `System.SerializableAttribute` 和 `System.NonSerializedAttribute` 搭配 `System.Xml.Serialization.XmlSerializer` 使用時是沒有作用的。  

> 這個例子很有感，多數專案都能看到這種誤用，這些沒作用的 `SerializableAttribute` 和 `NonSerializedAttribute` 真的很礙眼，又很難說服老闆開單拔這種單純礙眼的東西，最後就只能放著繼續礙眼。  

+ **X 避免** 把為了進階功能設計的型別和給基礎功能用的型別放在同一個命名空間下，這樣使用者比較容易快速了解並入門這個框架。

    JEFFRY RICHTER 表示建議將進階功能放在基礎功能的子命名空間，例如： `System.Mail` 是放基礎功能，`System.Mail.Advanced` 是放進階功能。

+ **X 不應** 建立沒有命名空間的型別。
    > 作者群補充了不少關於命名空間和編譯器與 CLR 之間的關係與運作基轉來附註說明這個原則，內容有點多且這個原則也很理所當然，所以就不記錄了。

### 預設子命名空間 (Standard Subnamespace Names)
少用型別應該放在子命名空間中避免弄亂主要的命名空間，這邊有一些預先定義好的子命名空間。

#### `.Design` 子命名空間
設計時期專用的型別應該放在這個子命名空間中，例如: 
``` csharp
System.Windows.Forms.Design
System.Messaging.Design
System.Diagnostics.Design
```

> 印象中這個在 ASP.NET Web Form 中很常見

#### `.Permissions` 子命名空間
權限相關的型別。

KRZYSZTOF CWALINA 表示他們有處理 `.Design` 命名空間，但是沒時間處理 `.Permissions`。例如：很多 `System.Diagnostics` 命名空間下與安全性的基礎建設相關的少用型別。

> 難怪沒什麼印象

#### `.Interop` 子命名空間
很多框架需要和舊的元件互動，互動相關的型別就可以放在這下面。

> 看起來像是和舊元件隔離層，讓上層能不用去和舊元件戶動。但是這邊太多看不懂的東西了，實在不知道怎麼整理，先留個 {% asset_link "PIA.pdf" "ChatGPT 的問答紀錄" %} 之後再回來看。

## 選擇使用類別或是結構
要做出適當選擇就要先了解他們之間的差異。  

首先是記憶體配置與回收，類別是值型別所以記憶體配置在堆積 (Heap) 中且受到垃圾回收 (GC) 機制管理與回收記憶體，而結構則不同，他可能配置在堆疊 (Stack) 中，也可能因為被其他型別包含而隨著包含著他的物件被配置在堆疊或堆積中，回收時機則是在堆疊回溯 (unwind) 時或是包含著他的物件被回收時。  

**一般來說在資源配置與回收上，值型別的成本低於參考型別。**  

> 關於 Unwind，ChatGPT 的解釋：  
> In the context of programming and the execution of methods or functions, "unwind" refers to the process of returning control back to the calling method or function and cleaning up any resources that were allocated during the execution of the called method or function. When the called method finishes execution, either by reaching its end or through an explicit return statement, the runtime needs to restore the state of the calling method and deallocate any resources that were allocated during the execution of the called method.

而陣列的部分，參考型別陣列是一連串的指標存在陣列中並各自指到實際儲存參考型別的位置上 (out-of-line allocation)，而值型別陣列就是他們本身的值 (inline allocation)，所以通常來說，值型別陣列在記憶體配置與存取上效能會好過參考型別陣列。  

CHRIS SELLS 表示，值型別有很多限制與不好用的地方，所以他比較推薦參考型別。使用值型別前提應該是知道型別被大量使用且有效能問題時再把參考型別改成值型別，而不是效能問題還沒發生就直接使用值型別。  

另一個部分是關於裝箱 (Boxing) 與拆箱 (Unboxing)，值型別在轉換成參考型別或其實作的介面時會將資料裝箱轉移到堆積中，轉型回值型別時拆箱轉移回堆疊中，頻繁裝箱與拆箱會對效能造成一定的負面影響，但參考型別之間的轉換就不會有這些問題。  

接著是指派 (Assignment) 與參數傳遞時，值型別都是整個物件複製，所以當值型別體積很大的時候就會佔用大量記憶體，反之參考型別傳遞的是物件的位置則沒這個疑慮。

就結論來說，通常使用類別，除非很必要需要結構的特性來滿足某些目的時才使用。

> 值型別和參考型別的差異以及他們各自的特性與注意事項在書中提到不少，但是有點細了所以沒有全部列出來，如果有空寫一篇關於值型別與參考型別的比較時再來仔細記錄。

+ **O 考慮** 在物件生命週期很短且通常依附在其他物件下時，優先使用結構。
    > 如果仔細了解值型別和參考型別的特性與記憶體使用方式的話，不難得到這個結論。

+ **X 避免** 使用結構，除非滿足以下所有情境：
  - 像 `int`、`float` 等原生型別一樣表達的是一個值。
  - 物件小於 16 bytes。
    > 這邊勞煩 ChatGPT 出面解釋一下 {% asset_link "16BytesStruct.pdf" "為什麼是 16 bytes" %} 了。
  - 不可變 (Immutable)。
  - 不常裝箱。

ERIC GUNNERSON 表示不需要執著於避免結構大於 16 bytes，而是根據型別的使用情境去分析要使用多大的值型別。

## 選擇使用類別或是介面
介面有一個很大的缺點是擴充性很差，在對外公布後再去增加介面的成員都會使得所有實作該介面的類別無法被編譯，所以介面一旦公開就難以改動了，如果要在不造成破壞的前提下增加成員就需要開另外一個新的介面。
> C# 8 中的預設介面方法 (Default Interface Method) 的出線，其中一個原因就是為了降低已存在的介面擴充時的困擾 ([Issues 52](https://github.com/dotnet/csharplang/issues/52)、[Proposal](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-8.0/default-interface-methods.md))。  
> 但就我個人觀點來看，預設介面方法提供的擴充性比較像沒有辦法的辦法，最好還是在設計介面時就考慮清楚。

**結論：一般來說，比較推薦使用類別來提供抽象功能。**  

+ **O 應該** 優先使用類別而非介面。
+ **O 應該** 使用抽象類別取代介面來解偶。

KRZYSZTOF CWALINA 表示介面即合約 (interface-as-contract) 的觀點是危險的迷思，會導致要將合約和實作分開時做出錯誤的決定。介面在將語法從實作中分離並沒有提供太大的幫助，合約本質是語意，而實作也能很好的表達語意。  
> 原文說的很抽象，理解起來有點障礙，所以一併抄錄 ChatGPT 的整理結果來幫助理解：  
> |  
> 【這個觀點對於軟體設計中經常被提及的「介面即合約」觀念提出批評。  
> 作者認為，介面主要用於定義使用物件所需的語法，但並未提供真正合約所需的語義。  
> 在軟體工程中，合約是一組規則和期望，詳細說明軟體應如何運行、交流和與其他組件互動。合約不僅包括語法，還包括語義，即代碼背後的含義或預期行為。  
> 作者認為，將介面視為合約的觀念會給人一種虛假的正確工程實踐感。這個觀點認為，語義（代碼的基本含義和規則）才是真正的合約，而它們實際上可以很好地與實現相結合。  
> 總之，這個觀點質疑介面在軟體設計中單獨定義合約的流行觀點。它強調語義在創建有意義的合約中的重要性，並批評將語法與實現分離的重點作為一種較低價值的工程實踐。】

KRZYSZTOF CWALINA 表示他跟團隊內一些人分享過這個看法，多數人曾經後悔將一些 API 透過介面開放出去。但沒有人說後悔開放類別。

CHRIS ANDERSON 表示如果太嚴格遵守這個規範，反而會陷入窘境。抽象類別雖然比較利於後續版本的發展與擴充，但使用後會使得類別唯一的繼承機會被消耗掉。介面適合用來定義兩個物件間不變的合約。抽象類別更適合用來定義多個類別間的共同父類別。
> 我的理解是介面適合被多個不同種類的類別實作且是明確不變的規格；而抽象類別則是會被多個相同種類的類別繼承。

JEFFREY RICHTER 表示繼承關係比較像是子類別是一種 (is-a) 父類別，而實作介面則像是類別能做到 (can-do) 介面定義的事。例如：`FileStream` 是一種 `Stream` 而它能釋放資源 (`IDisposable`)。

+ **O 應該** 在值型別需要多重繼承的情境時使用介面。  

值型別不能繼承任何型別，但可以實作介面。例如：  
``` csharp
public struct Int32 : IComparable, IFormattable, IConvertible
{
  ...
}
```

RICO MARIANI 表示好的介面通常都有這種混用 (mix in) 感，就是它不會只定義來給特定子類別實作的，更像是一種特性。

BRIAN PEPIN 表示另一個好的介面的特徵是介面只表達一件事，如果有一個介面有一堆功能那會是個警訊，之後很容易又會需要增加方法到介面中但卻無法辦到。
> 就之前說的，介面加方法會造成破壞性變更。  

+ **O 考慮** 用介面來達到類似多重繼承的效果。

## 抽象類別的設計
+ **X 不應** 把抽象型別的建構子設成 `public` 或 `protected internal`。 **O 應該** 把抽象型別的建構子設成 `protected` 或 `internal`。  

  原因是不能透過建構子直接建立抽象類別，所以 `public` 和 `protected internal` 的建構子會誤導使用者。
  > 有一點我覺得很奇怪，如果 `protected internal` 不適當，那為什麼同樣能被使用者存取到的 `protected` 就適當了?  
  >
  > 於是我跟 ChatGPT 聊了一下，得到的說法大概是 `protected internal` 比較複雜不容易一眼看透，相對容易混淆使用者，但 `protected` 比較明確。  
  >
  > 我是認為 ChatGPT 的說法有點遷強。  
  >
  > 不過回到背後原理：如果使用者技術上不能操作，就不要讓他看到然後誤用後才發現編譯不過。只要基於這個原理，要使用 `protected internal` 建構子也是可以接受的，當然 C# 7.2 增加的存取範圍較狹窄的 `private protected` 也理應適合使用於抽象類別的建構子。
  >
  > 附上 {% asset_link "ProtectedConstructorsInAbstractClass.pdf" "和 ChatGPT 的對話紀錄" %} 做為參考。

**O 應該** 至少提供一個類別繼承自抽象類別。
  這有助於驗證抽象類別的設計。

BRAD ABRAMS 表示他見過很多次精心設計過的抽象類別或介面在首次讓使用者使用後才發現不好，但已經來不及改了。  

CHRIS SELLS 表示根據他的經驗法則，至少要有三個人依照預期的情境實際使用後都能很漂亮的完成，這個設計才是好的。  

> 看來還是不要鐵齒。

## 靜態類別的設計

**O 應該** 謹慎使用靜態類別。
  靜態類別應該只能用在支援類/工具型的類別上，像是用來做一些"捷徑"來輔助主要功能的開發。

+ **X 不應** 把靜態類別當成雜物櫃把雜七雜八的功能都塞進去。
  > 好像曾經看過一個專案，裡面有一個 `Helper` 類別，然後什麼亂七八糟的輔助功能都放在裡面。

> 其他還有一些建議，但是如果使用 C# 的話這些規則都不可能違反(因為無法編譯成功)，所以這邊就不提那些規則了。

## 介面的設計

+ **O 應該** 當需要提供共用 API 且是用於結構 (Struct) 時使用介面來達成。  
+ **O 考慮** 在已經有繼承其他類別的時候提供介面來提供共用功能。 
  > 但我還是傾向遵守 JEFFREY RICHTER 的建議，也就是繼承是 is-a 的關係，而介面則是 can-do 的關。不管是因為什麼原因而使用介面，設計時仍然要符合這樣的定義。

+ **X 避免** 使用標記用的介面，也就是沒有成員的空介面。  
  如果需要的話應該使用 `Attribute` 來達到標記的效果。  

  RICO MARIANI 表示的確偵測 Attribute 比確認介面型別的成本還要高很多，所以基於效能還是可能會需要標記用的空介面，只是以他自己的經驗來說這種事不常見。

  這個想法的問題在於 `Attribute` 只能在執行階段確認，有時候必須在編譯時期就確認型別時就可以接受使用標記用的型別，例如下面的範例:  
  ``` csharp
  // empty interface
  public interface ITextSerializable {}
  
  public void Serialize(ITextSerializable item)
  {
    // use reflection to serialize all public properties
  }
  ```
  > 總之就是謹慎且只在必要時使用，所以才是 "避免" 而不是 "不應" 使用，例如 `IQueryable<T>` 就是空介面。

**O 應該** 至少提供一個類別實作設計好的介面。   
  和設計抽象類別一樣，這有助於驗證介面的設計。  

**O 應該** 至少提供一個 API 是有使用到設計好的介面的，"使用" 可以只是作為參數型別或屬性的型別。  
  也是為了驗證介面的設計。  

**X 不應** 在已發布出去的介面上新增成員。  
  這樣實作的類別會壞掉，所以只能建立新的類別來避免升版造成的問題，這也是之前提到應該優先使用抽象類別而不是介面。
  > 印象中 C# 8 新增的預設介面實作就是為了解決這個問題，但這個功能帶來了不少副作用和陷阱，有點複雜，這邊先記一下 {% asset_link "DefaultInterfaceMethodsProblems.pdf" "ChatGPT 的說明" %} 有機會再深入研究。

## 結構的設計

**X 不應** 設計可變的值型別(mutable value types)。  
有幾個原因：  
1. 值型別賦值時是是建立另外一個物件而不是原物件位址，開發者可能不會注意到他們修改屬性的時候修改的內容已不是原物件。
  > 如範例：  
    ``` csharp
    public struct Point
    {
        public int X { get; set; }
        public int Y { get; set; }
    }
    void ChangeX(Point p)
    {
        p.X = 3;
    }
    // Caller
    var point = new Point { X = 1, Y = 2 };
    ChangeX(point);
    
    // point.X is still 1
    point.X.Dump();
    ```
  > 
  > 不只是引數傳遞時，同樣的，包裹在參考型別中的值型別也會有一樣的問題如下：
    ``` csharp
    public class Container
    {
        public Point P { get; set; } = new Point() { X = 1, Y = 2 };
    }
    
    // Caller
    var c = new Container();
    var point = c.P;
    point.X = 3;
    
    // The value is still 1
    c.P.X.Dump();
    ```
  > 
  > 另外，雖然下面的程式碼是可以如預期運作的，但有鑑於上面情境下容易誤用，因此仍不應以此為理由來支持可變的值型別。
    ``` csharp
    // Caller
    var p = new Point(){ X = 1, Y = 2};
    p.X = 3;
    
    // Although it matches expectations, it is still bad because it is prone to being misused in othescenarios.
    p.X.Dump();
    ```

1. 有些語言 (尤其是動態語言)，即使只是取消參考 (dereference) 也會造成整個物件的複製。
  > 用 C# 會編譯失敗，這邊只為了示範，例如：  
    ```csharp  
    public class Container
    {
        public Point P { get; set; } = new Point() { X = 1, Y = 2 };
    }
    
    // Caller
    var c = new Container();
    // Dereference here.
    c.P.X = 3;
    
    // The value is still 1
    c.P.X.Dump();
    ```
  > 而正確的設計，至少要把屬性改成唯讀屬性 (只有 getter 沒有 setter)。
  > 例如：
    ``` csharp
    public struct Point
    {
        public Point(int x, int y)
        {
            X = x;
            Y = y;
        }
    
        public int X { get; }
        public int Y { get; }
    }
    ```


**O 應該** 確保所有結構內的值為 `0`、`false` (如果需要的話，包含 `null`) 時都是合法的。  
  > 所有值型別的預設值都是 `0`，差別只是如何展示資料，例如： `TimeSpan`，`DateTime`，`KeyValuePair<TKey, TValue>` 內的所有欄位，而 `bool` 的預設值 `false` 其實值也是 `0`。  

這是因為值型別的陣列初始化時的內部欄位的預設值都是 `0` 或是 `null` (如果欄位是參考型別)，而型別的陣列初始化時有參數的建構子並不會被執行。  
> 如範例：
  ```csharp
  public struct Percentage
  {
      public int Value { get; }
      
      public Percentage(int value)
      {
          if (value <= 0 || value > 100)
          {
              throw new ArgumentException("Percentage must > 0 and <= 100", nameof(value));
          }
  
          Value = value;
      }
  }
  
  var percentages = new Percentage[1];
  
  // This will succeed, but percentages[0].Value will be 0,
  // even though we might consider it an invalid state according to the constructor.
  percentages[0].Value.Dump();
  ```

  正確的設計應該要確保預設值是合法的。  
  > 例如 (有參考型別的欄位時依此類推)：
    ``` csharp
    public struct Percentage
    {
        private int _value;
        public int Value  => _value + 1;
        
        public Percentage(int value)
        {
            if (value <= 0 || value > 100)
            {
                throw new ArgumentException("Percentage must > 0 and <= 100", nameof(value));
            }
    
            _value = value - 1;
        }
    }
    
    var percentages = new Percentage[1];
    
    // It's now default to the minimal valid value 1.
    percentages[0].Value.Dump();
    ```

> 還真沒想過值型別陣列能這樣出問題...

+ **O 應該** 讓所有值型別實作 `IEquatable<T>`。  
  呼叫 `Object.Equals` 方法會造成裝箱 (boxing) 且他預設的實作方式因為使用了反射所以不太有效率。如果實作 `IEquatable<T>` 就能讓比對因為避免裝箱而更有效率。關於 `IEquatable<T>` 的實作在第 8 章第 6 節有說。  
  
  > 這邊要注意的是實作 `IEquatable<T>` 以避免裝箱是因為可以 **呼叫 `IEquatable<T>.Equals()` 來避免裝箱** 而不是呼叫 `Object.Equals()` 時避免裝箱，我在實驗過程一直無法重現還以為是書寫錯了，結果是我誤會。

  > 這邊附上範例程式，把程式碼透過 ILSpy 或其他工具轉成 IL 碼就可以看出有沒有裝箱的差別了。
  ``` csharp
      public struct Point
      {
          public Point(int x, int y)
          {
              X = x;
              Y = y;
          }
  
          public int X { get; }
          public int Y { get; }
      }
  
      public struct Point2 : IEquatable<Point2>
      {
          public Point2(int x, int y)
          {
              X = x;
              Y = y;
          }
  
          public int X { get; }
          public int Y { get; }
  
          public bool Equals(Point2 other)
          {
              return X == other.X && Y == other.Y;
          }
  
          public override bool Equals(object obj)
          {
              if (obj is Point2)
              {
                  return false;
              }
  
              return Equals((Point2)obj);
          }
      }
  
      public class Demo
      {
          public void Do()
          {
              var p1 = new Point(1, 1);
              var p2 = new Point(1, 1);
              // Box both p1 and p2.
              object.Equals(p1, p2);
              // Call ValueType.Equals(object?), box p2.
              p1.Equals(p2);
  
              var p3 = new Point2(1, 1);
              var p4 = new Point2(1, 1);
              // Box both p3 and p4.
              object.Equals(p3, p4);
              // Call IEquatable<T>.Equals(), without boxing.
              p3.Equals(p4);
          }
      }
  ```

另外，結構雖然有用但他只適用於小型、單一、其值具有不變性且不會頻繁裝箱的。
> 根據 ChatGPT 的說法，"單一" 指的是封裝單一或極少量的欄位值，總之就是盡可能簡單且單純。

## 列舉的設計

+ **O 應該** 在需要一組值的時候將參數、屬性定為一個回傳單一值的列舉。  
  > 舉例來說：
    ``` csharp
    [Flags]
    public enum DaysOfWeek
    {
        None = 0,
        Monday = 1,
        Tuesday = 2,
        Wednesday = 4,
        Thursday = 8,
        Friday = 16,
        Saturday = 32,
        Sunday = 64
    }
    ```
  > 上面的列舉可以在需要的時候將參數或屬性定義成屬性 `DaysOfWeek WorkingDaysOfWeek { get; set; }` 而他的值 31 代表周一到週五五天。  

+ **O 應該** 優先考慮列舉而不是靜態常數 (Static Constants)。  

JEFFRY RICHTER 表示列舉優先的設計能在搭配編譯器和反射的時候能得到更的方便。  

> 雖然列舉搭配 IDE 的自動提示和編譯時的強型別驗證通常能有比較好的開發體驗，但不是絕對的，如果評估後發現常數明顯比較適合時就不需要硬要用列舉 (實務上遇到過但情境太複雜特殊所以就不解釋了)。

+ **X 不應** 把開放性的集合 (Open Sets) 定義成列舉，例如作業系統的版本或是朋友的名字。
  > 應該是因為這些值是不是固定的，一旦需要增加就需要改程式並重新編譯程式碼，這類型的資料定義成字串或陣列等型別會比較穩定 (畢竟是對於框架設計的建議所以比較嚴格)。

+ **X 不應** 為了以後可能會用到預先定義列舉的保留值。
  > 例如：
    ``` charp
    public enum FoodType
    {
        Pizza,
        Burger,
        Pasta,
        Sushi,
        // 不應該出現的保留值
        Reserved1,
        Reserved2
    }
    ```

+ **X 不應** 開放只有一個值的列舉。
  > 其實就是 "不要建立保留選項" 的延伸，本意應該是不要建了一個列舉然後裡面 "只有一個預設值"。
  > 例如：
    ``` charp
    public enum FooType
    {
        Default,
    }
    
    public void Bar(FooType options)
    {
        //
    }
    ```
  > 
  > **如果是有意義的值應該不在此限。**

+ **X 不應** 在列舉中建立哨兵值 (Sentinel Values)。  
  以下面為例 (主要是因為這樣雖然方便但會混淆使用者)：  
  ``` charp
  public enum DeskType
  {
      FirstValue = 1, //  哨兵值

      Circular = 1,
      Oblong = 2,
      Rectangular = 3,

      LastValue = 3, // 哨兵值
  }

  public void OrderDesk(DeskType desk)
  {
      if (desk > DeskType.LastValue || desk < DeskType.FirstValue)
      {
          throw new ArgumentOutOfRangeException(...);
      }
  }
  ```

  建議的做法是如下面這樣：  
  ``` charp
  public enum DeskType
  {
      Circular = 1,
      Oblong = 2,
      Rectangular = 3,
  }

  public void OrderDesk(DeskType desk)
  {
      if (desk > DeskType.Rectangular || desk < DeskType.Circular)
      {
          throw new ArgumentOutOfRangeException(...);
      }
  }
  ```

RICO MARIANI 表示在使用列舉時太自作聰明會有很多麻煩。像是哨兵值這種東西，雖然用 `DeskType.LastValue` 來驗證然後之後 `DeskType.LastValue` 被更新的時候就驗證部分會 "自動生效" 好像很棒，但相對的使用者就會不知道要考慮新增的列舉值，而新增的值 4 就會變成沒被驗證擋下又沒被適當處理。

+ **O 應該** 提供預設值 0 作為列舉的預設值。
  某個值不適合作為列舉值時考慮把他定義成 `None` 之類的，最常見的就是設為 0，例如：
  ``` csharp
  public enum Compression
  {
      None = 0,
      GZip,
      Deflate,
  }
  ```

  > 我認為是因為值型別預設值是 0, 如果 0 有意義的話呼叫端其實很難判斷這個情境是沒處理好所以是預設值還是真的想要使用 0。因此將 0 定義成 "不適用" 是非常洽當的慣例。

+ **O 考慮** 使用 `Int32` 作為列舉預設的基礎型別 (Underlying Type)，除非：  
   + 預期會超過 32 個旗標 (Flag)。
   + 和非託管程式 (Unmanaged Code) 互動時需要不同大小的列舉。
   + 效能敏感而需要使用更小的型別的情境，例如：非常頻繁的被初始化、長度超長的集合、序列化大量資料等。

BRAD ABRAMS 表示發布後改列舉基礎型別是[二進位中斷性變更 (Binary Branking Change)](https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/breaking-changes#binary-breaking-change) 所以要謹慎考慮未來來做決定。以他們的經驗來說 `Int32` 通常都是正確的選擇所以他們才會用其作為預設。  

> 基本上就是用預設，除非是非常必要且不會再次變更的時候再考慮用其他基礎型別。

+ **X 不應** 直接擴充 `System.Enum`。
  他是 CLR 為了建立自定義的列舉 (User-defined Enumerations) 而提供的特殊型別，多數語言都由提供相關的元素來讓我們直接使用，例如：C# 中的 `enum`。

### 旗標列舉 (Flag Enum) 的設計 
+ **O 應該** 在旗標列舉上加上 `System.FlagsAttribute`。
+ **X 不應** 在一般列舉上加 `System.FlagsAttribute`。
  > 實測 C# 旗標列舉加上 `System.FlagsAttribute` 後如果加一個不是 2 的次方的列舉值竟然能編譯成功，這樣如果有人亂加破壞旗標列舉其實超容易也很危險...

+ **O 應該** 使用 2 的次方作為旗標列舉值，這樣才能順利使用位元運算 (Bitwise Operation)。  
  > 關於旗標列舉值, 當數量大的時候可以改個寫法就不用在那邊計算 2 的 N 次方是多少了，如下：
    ``` csharp
    [Flags]
    public enum ContentPermissions
    {
        None              = 0,
        ReadArticles      = 1 << 0,
        WriteArticles     = 1 << 1,
        DeleteArticles    = 1 << 2,
        EditArticles      = 1 << 3,
        PublishArticles   = 1 << 4,
        ReadVideos        = 1 << 5,
        UploadVideos      = 1 << 6,
        DeleteVideos      = 1 << 7,
        EditVideos        = 1 << 8,
        PublishVideos     = 1 << 9,
        ReadPodcasts      = 1 << 10,
        UploadPodcasts    = 1 << 11,
        DeletePodcasts    = 1 << 12,
        EditPodcasts      = 1 << 13,
        PublishPodcasts   = 1 << 14,
        ReadComments      = 1 << 15,
        WriteComments     = 1 << 16,
        DeleteComments    = 1 << 17,
        EditComments      = 1 << 18,
        ApproveComments   = 1 << 19,
        ManageUsers       = 1 << 20,
        BanUsers          = 1 << 21,
    }
    ```

+ **O 考慮** 使用一個單獨的列舉值來表市常用的組合。  
  例如：
  ``` csharp
  [Flags]
  public enum FileAccess
  {
      Read = 1,
      Write = 2,
      ReadWrite = Read | Write,
  }
  ```
  > 當然要合理， ReadWrite 就是 Read 加 Write，不是亂組阿。

+ **X 避免** 把某些組合不合理的情境設計成旗標列舉。  
  > 書上範例比較像是避免把不同概念合併成一個更抽象的列舉導致合理使用時編譯器卻出現警告，**而不是任意組合都要是合理的**，但範例說明很複雜我懶得打...  
  > 
  > 除了上述合併概念的情境外，一般情況下我的偏好是只要確定量不大就會預設做成旗標列舉，雖然列舉值不會同時成立，但是在用來判斷 "包含" 的時候是很好用的。
  > 以下面為例，雖然車輛不會同時是摩托車 (`VehicleType.Motorbike`) 又是腳踏車 (`VehicleType.Bicycle`) ，但是在用來判斷 "包含" (這台車是不是兩輪車) 的時候也很好用，例如：  
    ``` csharp
    [Flags]
    public enum VehicleType
    {
        None      = 0,
        Car       = 1,
        Motorbike = 2,
        Bicycle   = 4,
        Truck     = 8,
        TwoWheelers = VehicleType.Motorbike | VehicleType.Bicycle,
    }
    
    public bool IsTwoWheelersVehicle(Vehicle vehicle)
    {
        return VehicleType.TwoWheelers.HasFlas(vehicle.Type)
    }
    ```
  > 
  > 甚至要用在搜尋的情境也比集合很有效率，例如：
    ``` csharp
    // Use flag enum and bitwise operation for better performance.
    public bool IsDiscount(VehicleType discountVehicleTypes, Vehicle vehicle)
    {
        return discountVehicleTypes.HasFlas(vehicle.Type);
    }
    
    // Instead of use general enum and array with higher cost.
    public bool IsDiscount(VehicleType[] discountVehicleTypes, Vehicle vehicle)
    {
        return discountVehicleTypes.Contains(vehicle.Type);
    }
    ```

+ **X 避免** 使用 0 作為旗標列舉值，除非這個值代表的是 "以清除所有旗標" 且妥善命名。
  > 主要是位元運算時如果遇到 0 的話很多功能都會誤判，最簡單的就是 `foos.HasFlag(Foo.Zero)` 中，無論 `foos` 如何組成這個判斷的結果都是 `true`。其他位元運算可以仔細想想，應該不會只有這個情境。  
  >
  > 也因此我的偏好是直接把 0 定義成 None 表示不存在、不要用。 且 0 還是值型別的預設值，不要去用他會省去很多麻煩和後遺症。

+ **O 應該** 在列舉中把 0 定義成 `None`，對旗標列舉來說這個值一定要是 "已清除所有旗標"。  

### 新增列舉值

+ **O 考慮** 新增列舉值，即便有少許相容性風險。
  > 就是要加可以，但是要知道會有相容性的風險，如果足夠了解列舉應該能認知到可能的相容性風險，這裡就不贅述細節了。  
  > 反之當作為使用者的時候，在操作列舉時也要考慮列舉值增加後的相容與正確性。

## 巢狀型別 (Nested Types) 的設計

+ **O 應該** 在希望或必須讓內層類別需要存取外層類別的私有成員時使用巢狀型別。
  例如：
  ``` csharp
  public OrderCollection : IEnumerab1e 
  {
      Order[] data = ...;
      public IEnumerator<0rder> GetEnumerator()
      {
          return new OrderEnumerator(this);
      }
      // This nested type will have access to the data array
      // of its outer type.
      class OrderEnumerator : IEnumerator<0rder>
      {
      }
  }
  ```
  另外因為很多語言都支援 `foreach` 所以使用者幾乎不需要自己實作 enumerator。

  > 附上 ChatGPT 完成的程式碼如下 (用於概念示範所以不考慮細節的正確性)：
    ``` csharp
    public class OrderCollection : IEnumerable<Order> 
    {
        private Order[] data;
    
        public OrderCollection(Order[] orders)
        {
            data = orders;
        }
    
        public IEnumerator<Order> GetEnumerator()
        {
            return new OrderEnumerator(this);
        }
    
        class OrderEnumerator : IEnumerator<Order>
        {
            private OrderCollection _collection;
            private int _currentIndex;
    
            public OrderEnumerator(OrderCollection collection)
            {
                _collection = collection;
                _currentIndex = -1;
            }
    
            public Order Current => _collection.data[_currentIndex];
    
            public bool MoveNext() 
            {
                _currentIndex++;
                return _currentIndex < _collection.data.Length;
            }
    
            // Other IEnumerator methods are omitted for brevity
        }
    
        // Other IEnumerable methods are omitted for brevity
    }
    ```
  > 
  > 我實務上會用到巢狀型別的情境還有複雜的 Model，有些巢狀的 Model 的內層型別很特規完全無法共用，這時候把它收在內層能讓維護人員認知到內層型別只適用於該外層，範例如下：  
    ``` csharp
    public class Order
    {
        public string OrderId { get; set; }
        public List<ProductData> Products { get; set; }
    
        // ProductData is nested within Order and is used exclusively by it
        public class ProductData
        {
            public string Id { get; set; }
            public string Name { get; set; }
            // The Quantity in ProductData make sense only if ProductData for Order
            public int Quantity { get; set; }
        }
    }
    ```

+ **X 不應** 把巢狀型別用於 "分類"，應該要用命名空間來達到效果。
  > 上面的 Model 範例並不是把它用於分類。

+ **O 避免** 把巢狀型別公開 (Publicly Exposed)，除非在非常少見的情況下，它真的需要被其他呼叫端使用時。
  > 以上面 Model 的情境就是典型的例外，通常都要公開到 `internal` 甚至 `public` 層級，因為呼叫端賦值會需要，反序列化工具也很常要求型別要 `public`。

+ **X 不應** 把會被外部依賴的類別定義為巢狀型別。
  > 這邊的外部指的是 "包含著巢狀型別的主類別的外部"。

+ **X 不應** 將會由外部程式碼初始化的類別定義成巢狀型別，如果一個類別有公開級別的建構子 (Public Constructor)，那他就不應該是巢狀型別。
  > 但在應用層的專案中經常用到的資料容器型的類別 (例如：DTO)，即使會被外部程式碼初始化，只要定義上符合 "只適合它的外層類別" 我還是會考慮使用巢狀型別。

+ **X 不應** 將巢狀型別定義成介面的成員。

一般來說，巢狀型別使用上應該非常的保守，且避免將存取權限設為公開 (Public)。
  > 就我的理解，巢狀型別存在的情境是 "只適合它的外層類別"，所以可被存取範圍應該限縮在在外層類別內。  


## 型別與組件元資料 (Types and Assembly Metadata) 的設計
型別位於組件中，組件通常封裝成 `.dll` 或是 `.exe`，有些重要的特性 (attribute) 應該套用到包含公開型別的組件中，這一節關於這些特性的規則。  

+ **O 應該** 將特性 `CLSCompliant(true)` 套用到有公開型別的組件中。
  這個特性用來定義這個組件中的型別是相容於 CLS 的且可以被所有 .NET 支援的語言。例如 C# 中的 `uint` 不相容於 CLS 規範，C# 編譯時就會出現警告。
  > {% asset_link "CLSCompliantAttributeBenefits.pdf" "CLSCompliant Attribute Benefits from ChatGPT" %}

+ **O 應該** 將特性 `AssemblyVersionAttribute` 套用到有公開型別的組件中。

+ **O 考慮** 使用 `<主要版號>.<次要版號>.<編譯版號>.<修訂版號>` 的格式來訂定組件的版號，例如：3.5.21022.8。
  > {% asset_link "VersioningExplained.pdf" "Versioning explained from ChatGPT" %}

+ **O 應該** 將下列說明用的特性套用到組件中。這些特性會被像 Visual Studio 之類的工具用來提醒使用者關於組件的內容。  
  ``` csharp
  [assembly:AssemblyTit1e("System.Core.dll")]
  [assembly:AssemblyCompany("Microsoft Corporation")]
  [assembly:AssemblyProduct("Microsoft .NET Framework")]
  [assembly:AssemblyDescription(...)]
  ```

+ **O 考慮** 套用特性 `ComVisible(false)` 到組件中，而能被 COM 呼叫的 API 再另外開放。通常來說 .NET 組件不應該被 COM 看見，但如果 API 要讓 COM 呼叫的話可以對 API 或組件透過套用特性 `ComVisible(true)` 來開放。

+ **O 考慮** 套用特性 `AssemblyFileVersionAttribute` 和 `AssemblyCopyrightAttribute` 到組件中以提供更多關於這個組件的資訊。

> 平常碰的不夠底層這一節除的資訊揭露類型的部分外我是幾乎看不懂，雖然知道怎麼用，但就是不知道為什麼以及有什麼好處。

### 結論
這一章爆炸長的，拖了超久才終於整理完，但比起前幾章都算熟悉，這章中有發現不少自己不熟的東西也是好事。

### 參考
[Binary breaking change](https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/breaking-changes#binary-breaking-change)  

ChatGPT  
