---
title: Framework Design Guidelines 整理與心得 (5)
categories:
  - Reading
  - Framework Design Guidelines
tags:
---

這系列文章的目的是為了之後參考與快速回顧而基於 Framework Design Guidelines 這本書寫的整理與心得，這不是翻譯，所以內容都是閱讀理解後再梳理下來的，和書中語句必然不同，也只會保留我自已覺得需要的部分，並帶一些自己的想法與註解。

本篇包含了第五章的部分。 

<!--more-->

### 通用成員設計原則
#### 成員多載 (Member Overloading) 的設計

+ **O 應該** 使用明確的參數名字來表示較少參數的多載所使用的預設值。  
    ``` csharp
    public class Type
    {
        public MethodInfo GetMethod(string name); //ignoreCase = false
        public Methodlnfo GetMethod(string name, Boolean ignoreCase);
    }
    ```
    以上面例子而言，第二個多載應該命名為 `ingoreCase` (預設 false) 而不是 `caseSensitive` (預設 true)，因為 Boolean 的預設值是 false。  

    > 這個呼應了 Clean Code 觀點所建議的，讓參數命名符合其型別的預設值，尤其是 bool 更明顯，這樣可以清楚的知道各個參數的預設值就是型別的預設值，讓多載的維護與設計更容易。  
    > 當然很多情境沒辦法這麼理想，只能 "盡量"。

+ **X 避免**  讓不同多載的相同參數使用不同的命名。如果一個參數在兩個多載中都有出現，就應該使用相同的名字。  
    ``` csharp
    public class String
    {
        // correct
        public int IndexOf(string value) { ... }
        public int Indexof(string value, int startlndex) { ... }
        // incorrect
        public int Indexof(string value) { ... }
        public int IndexOf(string str, int startIndex) { ... }
    }
    ```

    > 這很好理解，就是整齊好維護。

+ **X 避免**  讓不同多載的相同參數們的排序不同，相同參數應該出現在相同位置。  
    ``` csharp
    public class EventLog
    {
        public EventLog();
        public EventLog(string logName);
        public EventLog(string logName, string machineName);
        public EventLog(string logName, string machineName, string source);
    }
    ```
    > 也是整齊好維護。

    但是有一些例外，例如使用 `params` 關鍵字的陣列參數必須在其他一般參數後面，當使用 `params` 關鍵字的陣列參數出現在多個多載中時，就需要取捨是要破壞參數順序還是不使用 `params`。  
    另外一個例外是使用 `out` 關鍵字的參數也是同理。  
    > 就是使用到需要固定參數位置的特性時就會遇到。

+ **O 應該** 只能讓擁有最多參數的成員作為虛擬成員使用 `virtual` 關鍵字。  
    而較少參數的成員們應該要簡單的呼叫下一個較多參數的多載就好。
    ``` csharp
    public class String
    {
        public int IndexOf(string s)
        {
            return Indexof(s, 9);
        }

        public int Indexof(string 5, int startIndex)
        {
            return Indexof(s, startIndex, s.Length);
        }
        
        public virtual int IndexOf(string 5, int startIndex, int count)
        {
            //do real work here
        }
    }
    ```

    BRIAN PEPIN 表示這個規則也可以套用到抽象方法中，在抽象類別中可以將所有必要的驗證都放在非抽象以及非虛擬方法中，只提供一個抽象方法給開發者去實作。

    CHRIS SELLS 表示這個規則除了比較好寫，測時時也不用測試所有的多載。另外如果有兩個多載，他總是讓較少參數的多載呼叫次少參數的多載而不是最多參數的多載，他不會在多個多載中重複填寫各參數的預設值，如果發生這種事可能就是寫錯了。

    > 總之就是：  
    > 1. 關於多個多載之間的呼叫順序，依序呼叫能將維護的時混淆的風險降到最低。  
    > 1. 關於參數驗證，建議放在最多參數 (同時也是主要實作) 的方法中。  
    > 
    > 雖然規則好像很多，但背後的概念很多都是為了**易於維護與測試**，就算不記這些規則，只要知道背後的目的，最後開發出來的程式也會往這些規則靠攏。  

+ **X 不應** 讓多載的成員使用 `ref` 或 `out` 修飾詞。  
    例如下面的錯誤範例：
    ``` csharp
    public class SomeType
    {
        public void SomeMethod(string name) { .. . }
        public void SomeMethod(out string name) { . . . }
    }
    ```
    一方面是有些語言在呼叫時無法辨識這兩種多載，另一方面是這種情況通常應該設計成兩個不同的方法。

    > 我原本認為多載用途這麼明確的東西不太可能誤用，直到看到某個專案把幾乎不同功能的幾個方法開成多載，然後讓方法名稱抽象到看不出具體功能，例如： `handleRequirement(...)`，才意識到現實的殘酷...  
    > 總之，多載就是給**相同功能**不同參數的情境用的。

+ **X 不應** 在多載中相同位置上放型別相近但意義不同的參數。  
    例如這個極端的例子：
    ``` csharp
    public class SomeType
    {
        public void Print(long value, string terminator)
        {
            Console.write("{e}{1}", value, terminator)g
        }
        public void Print(int repetitions, string str)
        {
            for (int i = 0; i < repetitions; i++)
            {
                Console.write(str);
            }
        }
    }
    ```
    而如果兩個參數語意上是相同的就可以，例如：
    ``` csharp
    public static class Console
    {
        public void WriteLine(long value) { . . . }
        public void WriteLine(int value) { . . . }
    }
    ```

    > 但我認為還是要盡量避免，尤其當兩個型別可以隱含轉型的時候，以下面 C# 範例來說：  
    ``` csharp
    public void WriteLine(long value) { "a".Dump(); }
    public void WriteLine(int value) { "b".Dump(); }
    int n = 1;
    WriteLine(n);
    ```
    > `WriteLine(n)` 會呼叫包含 `int` 參數的多載，但是當 `public void WriteLine(int value) { "b".Dump(); }` 被移除時，編譯還是會通過，但變成呼叫包含 `long` 參數的多載。  
    > 這對實務上的維護並不友善，會讓維護時的風險提高，很容易不小心產生意外而不自知。

+ **O 應該** 允許 `null` 作為預設的選擇性引數。  
+ **O 應該** 優先使用多載而不是預設引數。  
    > 這兩個規則高度相關，放在一起講。  
    如果選擇性引數是參考型別，用 `null` 作為預設可以降低呼叫端做空值判斷的麻煩。

    BRAD ABRAMS 表示這不是鼓勵將 `null` 作為 magic constant，而是用來避免額外的檢查，事實上當在呼叫 API 是傳遞 `null` 作為引數通常代表弄做了什麼或是 API 的多載設計不良。

    > 內部工具用選擇性參數來減少多載還滿實用的，但用在公開套件就不太適合了。主要是選擇性引數是編譯時期語法糖，編譯後自動幫呼叫端填入預設參數值，所以當套件 API 更新時呼叫端需要重新編譯，如果直接參考新的 dll 可能會造成 API 呼叫不到或是呼叫時傳入舊的預設值。  
    > 關於選擇性引數的問題和範例應該滿多的就不離題了，或是問問 ChatGPT 也能得到更詳細的說明。

#### 明確實作介面成員的設計 (Implementing Interface Member Explicitly)

+ **X 避免** 明確實作介面成員，除非有強烈的理由。  
    明確實作介面成員除了會讓呼叫端因為無法直接呼叫公開成員而疑惑之外，也可能造成不必要的裝箱 (Boxing)。  
    
    > 因為呼叫端呼叫明確實作介面的成員時，需要向上傳型成介面後才能呼叫，當該型別是值型別時就會在向上轉型時造成裝箱。  
    > 例如: `((IInterface)myStruct).ExplicitlyImplemented();` 就會在不得已需要轉型的過程造成裝箱。

    ANDERS HEJLSBERG 表示有些其他環境的開發者抱怨需要因為實作介面而將只提供內部呼叫的方法公開，這時候明確實作介面成員就是個正確的解決方案，但他不常聽到這樣的需求。

    RICO MARIANI 表示值型別通常很小型且只有非常簡易的操作。如果需求太簡單，那函數呼叫的成本可能超過需求本身。要注意值型別的本意是為了低執行成本 (Cheap)。

+ **O 考慮** 在必須轉型成該介面後在呼叫相關成員才合理的情況下明確實作介面成員。  
    這包含當該成員主要是提供框架內部使用時。例如 `ICollection<T>.IsReadOnly` 主要是讓底層與資料綁定 (Data-Binding) 相關功能透過 `ICollection<T>` 介面呼叫，對使用 `List<T>` 的情境來說幾乎用不到，所以 `List<T>` 明確實作 `ICollection<T>` 介面的 `IsReadOnly` 屬性，讓我們在使用 `List<T>` 時不會看到用不到的 `IsReadOnly` 屬性。

    > 只有一個範例很難體會，可以到 .NET 官方 API 網站去翻翻看其他範例。

+ **O 考慮** 在需要模擬變異性 (variance) 的情境時明確實作介面成員。  
    例如實作 `IList` 時，常需要透過明確實作介面成員來 "隱藏" 回傳 `object` 的成員，改用回傳特定型別的成員，如下範例：
    ``` csharp
    public clss StringCollection : IList
    {
        public string this[int index] {...}
        object IList.this[int index] {...}
    }
    ```

    > 這樣當使用 `StringCollection` 的索引子時就能直接得到字串型別的結果而不用額外處理型別轉換。

+ **O 考慮** 明確實作介面成員來隱藏介面成員並提供另一個更恰當命名的同功能方法。  
    ``` csharp
    // this is not exactly how FileStream does it but this simplification
    // best illustrates the concept
    public class FileStream : IDisposable
    {
        void IDisposable.Dispose() { Close(); }
        public void Close() { . . . }
    }
    ```
    這種做法應該要極度保守的使用，幾乎都是弊大於利。  

    > 這麼容易濫用的做法我是傾向乾脆不要用，不然之後轉手交接維護幾次保證歪掉。

+ **X 不應** 明確實作介面成員來作為安全邊界 (Security Boundary)。  
    呼叫端只要向上轉型就能呼叫到原本的成員。

+ **O 應該** 在明確實作介面成員的功能是需要給衍生類別覆寫的情形下，用另外一個相同功能的成員來實作(`protected`、`virtual`)。  
    明確實作介面成員無法被覆寫，所以當該成員需要設計給衍生類別來覆寫時就會需要用另外一個可被覆寫的成員來實作，而明確實作介面成員就單純呼叫那個可被覆寫的成員，如下範例：  
    ``` csharp 
    [Serializable]
    public class List : ISerializable
    {
        void ISerializable.GetObjectData(Serializationlnfo info, StreamingContext context)
        {
            GetObjectData(info,context);
        }
        
        protected virtual void GetObjectData(SerializationInfo info, StreamingContext context)
        {
            // ...
        }
    }
    ```

#### 屬性與方法的選擇
一般來說，可以分成以方法為主 (Method-heavy，透過方法存取欄位資料) 以及以屬性為主 (Property-heavy，透過屬性存取資料) 兩種 API 設計風格。  

對於欄位資料是以方法還是屬性來操作，以屬性為主的設計通常是比較推薦的，不然資料多的時候方法參數會很多，對初階開發者不友善。 但方法為主的設計在效能上比較好，且設計出來的 API 對進階開發者來說可能比較好用。  
> 我覺得要看情境和脈絡，比較不像是面對初階還是進階開發者的差異。  

以經驗來說，方法用來提供 "行為" 而屬性提供的是 "資料"，如果其他條件 (效能、可維護性、可讀性等) 差不多時，推薦使用屬性。  
> 這一節比較像是在說 "操作資料/欄位" 這個議題上要選擇屬性還是欄位。  

RICO MARIANI 表示屬性操作上看起來很像欄位，呼叫端會直接存取他而不會希望存取屬性的效能比欄位差太多，同時也不會預期每次取得的屬性值不固定。

JOE BUFFY 表示屬性應該只包含少量程式，所以避免 `if` 條件式、try-catch、呼叫其他方法等，並且只包含存取欄位的內容，這樣就能避免 Rico 說的效能問題。  
> 這裡很重要，我們常常會不小心讓屬性的職責太多，但又不知道邊界在哪、怎樣算太寬、怎樣算適合...等等。  
>
> 統整來說：  
> 1. 固定的輸出結果。
> 1. 只做存取欄位的事。
> 1. 不做 (包含但不限於) 條件式、迴圈、例外處理等操作。

+ **O 考慮** 使用屬性來表示型別中扮演特性 (Attribute) 一角的成員。  
    例如： `Button.Colar` 設計成屬性是因為顏色是按鈕的一個特性。  
    > 或是說 "特徵" 比較貼切。

+ **O 應該** 使用屬性而非方法的情境有：  
    - 在當屬性值存在於程序的記憶體 (Process Memory) 中時，且屬性只用於取得該欄位值。  
        例如下面的範例，取欄位 `name` 的值時應該用屬性：  
        ``` csharp
        public Customer
        {
            public Customer(string name)
            {
                this.name = name;
            }

            public string Name
            {
                get { return this.name; }
            }

            private string name;
        }
        ```

        > 有自動實作屬性後這個範例在實務上不太會遇到，即使想將值設計成不可變 (Immutable)，在 C# 9 中也有提供關鍵字 `init` (Init Only Setters) 來輕易達到。  
        > 我比較意外的是原來放在堆積中的資料其實是存在於程序的記憶體中而不是執行緒中，之後要找時間看一下關於程序和執行緒的記憶體配置，以及 C# 的物件之類的內容怎麼被儲存。

+ **O 應該** 使用方法而非屬性的情境有：  
    - 當執行成本比欄位存取慢幾個量級時。例如：非同步行為、網路連線、檔案存取等。  
        > 除非是萬不得已，否則我不會讓屬性中出現任何邏輯。
    - 當該操作是轉換 (Conversion) 時。  
    - 當每次回傳結果都不同時，例如：`Guid.NewGuid()` 方法每次回傳值都不同。  
        JEFFERY RICHTER 表示 `DateTime.Now` 屬性因為每次回傳值都不一樣，其實應該要設計成方法。
    - 當該操作有明顯的副作用時。要注意的是操作內部快取 (Internal Cache) 不算副作用。  
        > 但我認為如果操作的快取是可能會被類別外的物件存取的話，就應該算是副作用而不應該使用屬性了。  

        BRAIN PEPIN 表示 Windows Form 控制項就是太多副作用的典型案例。存取 `Handle` 屬性會建立原本還沒建立的控制代碼 ([handle](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.control.handle))，經常因為在除錯過程因為讀取 `Handle` 屬性而改變了 debugging session，進而隱藏了 bug。  

        > Windows Form 我沒寫過，所以不太知道具體細節。但是類似於上面的案例我以前也遇過幾次，基本上就是因為在 IDE 中除錯時，可能會想停在中斷點時去觀察一些屬性的值，結果這個屬性除了回傳值外還去建立或改變一些物件、資料等，導致意外的 "改變" 干擾除錯過程，而我們還不一定會察覺到這個干擾導致最後一直找不到問題或是找錯問題。  
    - 當該操作回傳內部狀態的副本時 (A copy of an internal state)，但回傳值型別除外。  
        > 理由應該和下一點提到的回傳陣列的情境一樣。  
    - 當該操作回傳陣列時。  
        通常回傳內部陣列時需要複製後再回傳而不能直接回傳參考，這樣才不會被呼叫端改到內部狀態，但物件複製效能不好會違反屬性應該高效率的原則。  
        因應方式有兩種。一是改成方法，讓呼叫端鎮知道這個呼叫不是直接取得內部資料；二是回傳唯讀集合 (`ReadOnlyCollection<T>` 之類的) 型別的物件，避免被外部改變內部資料的同時又不需要複製陣列。  
        BRAD ABRAMS 表示這個回傳陣列的屬性所造成的問題是 .NET Framework 1.0 時候真實遇到的，數以千計的大量陣列都在建立後段時間內就廢棄回收，才因此訂立了這個規則。  

RICO MARIANI 表示這所有規則都很重要，能避免很多嚴重的問題。 整體可以簡化成一個口訣：**屬性用於單純存取簡單資料或簡單的運算**。 不要諞離這個核心。  


### 屬性設計原則

+ **O 應該** 在不允許呼叫端變更屬性值時使用唯讀屬性。  
    要注意的是即使是唯讀屬性，但如果屬性是參考型別，呼叫端還是可以改變屬性的內容。  
    > 這點應該是常識，但是參考型別的唯讀屬性內容可被變更這件事卻很容易被忽略，所以還是特別記一下。  
    > 如果真的要嚴格禁止呼叫端變更屬性值與內容，那就應該使用(或自製)唯讀型別，例如：`ReadOnlyDictionary<TKey,TValue>`、`ReadOnlyList<T>` 等。

+ **X 不應** 讓屬性的 setter 的存取範圍 (Accessibility) 比 getter 高。  
    像是 public setter 搭配 protected getter 就不好。如果真的需要則改以 `Set` 為前綴的方法來實作，例如：`AppDomain.SetCachePath(String)` 方法。  
    > 這我其實沒有很理解，而 ChatGPT 是這樣說的：  
    > The key difference lies in how properties and methods are commonly understood and used, both syntactically and semantically.  
    > 1. Semantic Expectations: Properties are generally understood to represent the state of an object. A set-only property or one with different access levels for the getter and setter breaks this expectation, which can lead to confusion. Methods, on the other hand, are expected to perform actions, so they don't come with the same expectation of symmetry between getting and setting.  
    > 2. Syntactic Uniformity: In languages like C#, the property syntax suggests that getting and setting are two sides of the same coin. When you have a set-only property or asymmetrical access levels, you're breaking this syntactic uniformity, which can lead to confusion and misuse. Methods don't carry this same expectation of uniformity.  
    > 3. Clarity and Discoverability: When you have a method to set a value, it's clearer that the method is doing something specific. This is especially true if the method has a descriptive name. A property with a public setter and a private getter doesn't offer this level of clarity and can be misleading. Developers might not easily realize that they can set but not get the property, leading to confusion.  
    > 4. Flexibility: Methods can be overloaded to provide multiple implementations, allowing for more flexible and descriptive APIs. Properties can't be overloaded in the same way, making them less flexible when you need to extend functionality.
    > 5. Parameterization: Methods can take parameters, providing additional context for the operation. This isn't possible with properties, which are parameterless by definition.
    > 6. Versioning: Methods are easier to evolve. If you decide to change how the value is set, or if you decide a getter should be added later, methods provide a more straightforward path for these kinds of changes without breaking the existing API.  
    > 7. Documentation: It is easier to document the exact behavior, limitations, and side-effects of a method compared to a property, particularly when the property has non-standard behavior like being set-only or having different access levels for the getter and setter.  
    >
    > In summary, while both properties and methods can encapsulate behavior, they are not perceived the same way. Using a method instead of a non-standard property provides more clarity, better adheres to the principle of least surprise, and offers more flexibility for future changes.  
    > 
    > 整體來說，比較多是環繞在語意和成員的設計概念上的差異。

+ **O 應該** 提供合理的預設值給屬性，並確保預設值沒有安全漏洞或效能低落的問題。  
    > 預設值的本意就是最常見的通用情境，通常是最被大量使用且使用者不會關心相關細節，當然必須安全且高效。  

+ **O 應該** 允許屬性可被依照任何順序賦值，即使當下物件狀態不正確。  
    > 我的標準會更嚴格，屬性賦值不應該有順序依賴性，不允許屬性賦值過程問題導致物件狀態不正確這種事發生。  
    > 以目前實務上遇過的大概幾種狀況：  
    > 1. 當多個屬性需要同時賦值時，把相關屬性 setter 的存取範圍改成 private 同時改用方法來提供多屬性同時賦值的功能，例如：  
    ```csharp
    public class Person
    {
        public string FirstName { get; private set; }
        public string LastName { get; private set; }

        public void SetName(string firstName, string lastName)
        {
            FirstName = firstName;
            LastName = lastName;
        }
    }
    ```
    > 2. 有些屬性不允許空值時，用建構子來強制呼叫端在建立物件時就填好必需有值的屬性，例如：  
    ```csharp
    public class Person
    {
        // Always be private.
        private Person()
        {
        }

        public Person(string firstName, string lastName)
        {
            FirstName = firstName;
            LastName = lastName;
            Age = 0;
        }

        public string FirstName { get; private set; }
        public string LastName { get; private set; }
        public int Age { get; set; }

        public void ChangeName(string firstName, string lastName)
        {
            FirstName = firstName;
            LastName = lastName;
        }
    }
    ```
  
+ **O 應該** 在 setter 拋出例外的情境下保留原本的值，意即賦值不能成功。

+ **O 避免** 在 getter 中拋出例外。  
    屬性取值是不應該會失敗的，如果真的需要拋出例外代表應該重構成方法。 但這個規則不是用於索引子 (Indexer)。

    PATRIC DUSSUD 表示這個規則不適用在 setter 上，在 setter 中拋出例外是可行的。

#### 索引子(Indexed Property) 的設計
> 書中標題用 Indexed Property (經查是 VB 的用詞) 而不是用 Indexer 表示，但內容都是使用 Indexer。  

RICO MARIANI 表示應該預期索引子會在迴圈中被呼叫，所以應該保持簡單。

+ **O 考慮** 使用索引子提供對內部陣列的操作。

+ **O 考慮** 在集合類型中提供索引子。

+ **X 避免** 在索引子中提供超過一個參數。
    如果需要多個參數就要想想他其實不是用來操作"集合"，如果不是就應該重構成方法。  
    > 不過如果不是用來表達集合，而是用來表達矩陣類的型別應該還滿適合的。

+ **X 避免** 讓索引子的參數在 `System.Int32`、`System.Inet64`、`System.String`、`System.Object`、列舉 (Enum) 之外。  

+ **X 不應** 提供和已存在的索引子一樣功能的方法。  
> 應該是理所當然的事，但不小心可能在將索引子重構成方法的過程中，會想要保留兩種呼叫方式所以還是記一下。

#### 屬性變更通知的事件 (Property Change Notification Events)

+ **O 考慮** 當 hight-level API 的屬性變更時發起變更通知事件。
    雖然是這樣建議，但是如果是 low-level 的 API (例如 `List<T>`) 就不應該因為屬性變更而發起事件。

+ **O 考慮** 當屬性值因外部因素變更時發起變更通知事件。
    屬性值因外部因素 (不是透過呼叫該物件的方法) 變更時，發起事件意味著能讓開發者知道其值改變了。例如 `TextBox` 控制項中的 `Text` 屬性在使用者輸入時就會自動改變。

> 在 Web 開發的情境幾乎沒遇到過，沒太大感觸，比較相似的就是發佈訂閱模式的 `IObservable<T>` 和 `IObserver<T>` 吧，使用時倒是能參考一下這一節。  

### 建構子的設計

+ **O 考慮** 提供簡單且合理的建構子。  
    簡單的建構子參數應該很少且都是原生型別或列舉，這樣能提高框架的易用性。  

+ **O 考慮** 當要做的行為在語意上不僅僅是建構一個實例，或如果遵守建構子設計建議會變得很不自然時，使用靜態工廠方法取代建構子。  
    > 描述很抽象，讓 ChatGPT 幫寫範例吧： 
    > Sure, let's break it down with a simple example for each of the two scenarios described:  
    > 1. Semantics of the operation do not map directly to the construction of a new instance:  
    Suppose we have a `Logger` class, and we want to provide a logger instance that could potentially be shared across different parts of the application. Here, the operation of getting a logger is not necessarily about creating a new instance; it's about obtaining an appropriate logger object.  
    ```csharp
    public class Logger {
        // Private constructor prevents direct instantiation.
        private Logger() { }

        // Static factory method that provides a singleton logger instance.
        public static Logger GetInstance() {
            if (_instance == null) {
                _instance = new Logger();
            }
            return _instance;
        }

        private static Logger _instance;
    }
    ```
    >   In this case, `GetInstance` does not always map to constructing a new `Logger`; sometimes, it retrieves an existing one, so the semantics are about instance retrieval rather than construction.  
    > 2. Following constructor design guidelines feels unnatural:
    > For a scenario where constructor guidelines feel unnatural, consider a `Temperature` class where we want to provide the ability to create a temperature object from degrees Celsius or Fahrenheit. Using constructors would be confusing because it's not clear which unit is used.  
    ```csharp
    public class Temperature {
        public double DegreesCelsius { get; }

        // Private constructor prevents ambiguity.
        private Temperature(double degreesCelsius) {
            DegreesCelsius = degreesCelsius;
        }

        // Static factory methods for clarity.
        public static Temperature FromCelsius(double degrees) {
            return new Temperature(degrees);
        }

        public static Temperature FromFahrenheit(double degrees) {
            return new Temperature((degrees - 32) * 5 / 9);
        }
    }

    // Usage
    var tempC = Temperature.FromCelsius(0);
    var tempF = Temperature.FromFahrenheit(32);
    ```
    >   Using static factory methods here clarifies the intended unit of temperature, whereas constructors would not. Instead of having constructors like `Temperature(double degrees)` and being unsure about the unit, we have `FromCelsius` and `FromFahrenheit` methods that clearly express their purpose.


+ **O 應該** 透過建構子參數來設定主要的屬性。   
    > 例如下面的範例，只要能初始化這個物件就一定是合理的：  
    ``` csharp
    var ron = new Person("Ron", Gender.Male);
    ```
    > 但是反例如下，初始化後還要做一些設定才能讓這個物件合理的話，這個型別就會很難使用：  
    ``` csharp
    var ron = new Person();
    // User unfriendly, caller should remember to initialize main properties after cosntructing an instance of Person.
    ron.Name = "Ron";
    ron.Gender = Gender.Male;
    ```

+ **O 應該** 最小化建構子要做的事。
    建構子除了把參數值存起來外不應該多做太多事，其他事延遲到真正需要的時候再做。  
    > 不過像前面的建議說的，如果是必要的行為還是要做，不然就改用靜態工廠方法。

+ **O 應該** 在適當的時候由實例建構子 (Instance Constructors) 中拋出例外。  
    CHRISTOPHER BRUMME 表示即使在建構子中拋出例外這個物件仍然有被建立，如果有實作解構子 (Finalize Method / Destructor) 的話仍然會在適合的時候被 GC 呼叫，也就是必須確保解構子能相容部分建構的物件。  
    > CHRISTOPHER BRUMME 的這個提示很重要，解構子會在 GC 過程被呼叫，如果在解構子中不相容建構子沒有正確執行完的情境而拋出例外的話，根據 ChatGPT 的說法是可能造成程序 (Process) 終止導致應用程式關閉的，這邊也提供官方資料 [Exceptions in managed threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/exceptions-in-managed-threads)、[How finalization works](https://learn.microsoft.com/en-us/dotnet/api/system.object.finalize#how-finalization-works) 作為參考。  

    JOE DUFFY 表示有些建構子的反模式也會展現出 Chris 提到的問題。 如果在建構子中過早分享 `this` 的參考，這個物件可能會在他完全被建構前被存取甚至於拋出例外。這些問題可以發生在設定 `this` 給參數物件的欄位或靜態變數等情境。不惜一切代價都應該避免這種事。  
    > 看起來就是花式挖洞給自己跳，對於這種情境我一時間想不到範例，就交給 ChatGPT 解決吧：  
    > The anti-pattern described here involves the misuse of the `this` reference in a constructor, which can lead to the object being accessible before it is fully constructed, especially if an exception is thrown during the construction process. Sharing the `this` reference prematurely can occur in various ways, such as passing `this` to a method or assigning it to a static variable within the constructor. Here are simple examples demonstrating this anti-pattern:
    > Example 1: Passing `this` to a method within the constructor  
    ```csharp
    public class EventPublisher
    {
        public event EventHandler<ObjectInitializedEventArgs> ObjectInitialized;

        public EventPublisher()
        {
            // Prematurely sharing 'this' reference by invoking an event
            ObjectInitialized?.Invoke(this, new ObjectInitializedEventArgs());
            // If an exception occurs here, the partially constructed object was already exposed
            throw new Exception("Failed to complete initialization.");
        }
    }

    public class ObjectInitializedEventArgs : EventArgs
    {
        // Event arguments details
    }
    ```
    > Example 2: Setting a field on an object passed in as an argument
    ```csharp
    public class SomeDependency
    {
        public object Owner { get; set; }
    }

    public class SomeClass
    {
        public SomeClass(SomeDependency dependency)
        {
            // Prematurely sharing 'this' reference by setting it on an external object
            dependency.Owner = this;
            // If an exception occurs here, the partially constructed object was already exposed
            throw new Exception("Failed to complete initialization.");
        }
    }
    ```
    > Example 3: Assigning `this` to a static variable
    ```csharp
    public class GlobalRegistry
    {
        public static SomeClass LatestInstance;

        public SomeClass()
        {
            // Prematurely sharing 'this' reference by assigning it to a static variable
            GlobalRegistry.LatestInstance = this;
            // If an exception occurs here, the partially constructed object was already exposed
            throw new Exception("Failed to complete initialization.");
        }
    }
    ```
    >In all these examples, if an exception occurs after the `this` reference has been shared but before the constructor completes, you have a partially constructed object that is accessible from other parts of the program. This can lead to unpredictable behavior and difficult-to-diagnose bugs. Therefore, it is important to avoid such patterns and ensure that the `this` reference is not exposed until the object is fully constructed and in a consistent state.

+ **O 應該** 在類別中明確寫出公開預設建構子 (如果需要的話)。
    如果沒有寫建構子的話 C# 編譯器會自動幫類別填入 `public` 建構子 (抽象類別則是 `protected` 建構子)，而當寫了有參數建構子時則不會自動填入。 這會導致維護時如果新增一個有參數建構子到類別中的話，則原本自動建立的預設建構子就不會建立，造成預期外的破壞性變更，而這樣的問題在程式碼審查時很難被發現。  
    這點不適用於結構，因為即使有其他建構子，編譯結構時仍然後預設加入預設建構子。

+ **X 避免** 在結構中明確寫出預設建構子。  
    這能讓陣列建立的更快，因為預設沒有寫預設建構子時陣列建立時就不用一個一個的呼叫結構的建構子(沒有預設建構子的結構，會由 CLR 提供空的預設建構子，CLR 在建立陣列時沒必要去執行)。  
    > 先說結論，這個建議不用關心。  
    > C# 10 之前是不允許建立結構的無參數建構子的所以不用關心這個問題，而 C# 10 支援結構的無參數建構子了，實測發現在 C# 10 下建立陣列時結構的無參數建構子也不會被執行，相當於這個疑慮已經不存在。  
    > 我們這邊可以用以下程式碼來觀察：
    ``` csharp
    public struct MyStruct
    {
        public static int InstancesCount = 0;

        public int X;
        
        public MyStruct()
        {
            InstancesCount++;
            X = -1;
        }

        public MyStruct(int x)
        {
            X = x;
        }
    }

    // From caller
    var myStructs = new MyStruct[1];
    MyStruct.InstancesCount.Dump(); // 0
    myStructs[0].X.Dump(); // 0
    ```

+ **X 避免** 在建構子中呼叫虛擬成員。
    呼叫虛擬成員會導致繼承鏈最底層的衍生類別在建構完成前，其覆寫的成員就已經被呼叫。  
    例如下面的範例，`Derived` 的建構子在被呼叫前，`Method()` 就被呼叫了，導致 `value` 值不如預期：      
    ``` csharp
    public abstract class Base
    {
        public Base()
        {
            Method();
        }

        public abstract void Method();
    }

    public class Derived : Base
    {
        private int value;

        public Derived()
        {
            value = 1;
        }

        public override void Method()
        {
            if (value == 1)
            {
                Console.WriteLine("All is good");
            }
            else
            {
                Console.WriteLine("What is wrong?");
            }
        }
    }
    ```
    > 範例是用抽象方法，我這邊提供另外一個更完整的範例來展示執行順序，且同時包含抽象和虛擬方法：
    ``` csharp
    public abstract class Base
    {
        public Base()
        {
            AbstractMethod();
            VirtualMethod();
            "Base constructed".Dump();
        }

        public abstract void AbstractMethod();

        public virtual void VirtualMethod()
        {
            "Base.VirtualMethod() called".Dump();
        }
    }

    public class Derived1 : Base
    {
        public Derived1()
        {
            "Derived1 constructed".Dump();
        }

        public override void AbstractMethod()
        {
            "Derived1.AbstractMethod() called".Dump();
        }

        public override void VirtualMethod()
        {
            "Derived1.VirtualMethod() called".Dump();
        }
    }

    public class Derived2 : Derived1
    {
        public Derived2()
        {
            "Derived2 constructed".Dump();
        }

        public override void AbstractMethod()
        {
            "Derived2.AbstractMethod() called".Dump();
        }

        public override void VirtualMethod()
        {
            "Derived2.VirtualMethod() called".Dump();
        }
    }

    public class Derived3 : Derived2
    {
        public Derived3()
        {
            "Derived3 constructed".Dump();
        }

        public override void AbstractMethod()
        {
            "Derived3.AbstractMethod() called".Dump();
        }

        public override void VirtualMethod()
        {
            "Derived3.VirtualMethod() called".Dump();
        }
    }

    new Derived3();
    /* Output:
    Derived2.AbstractMethod() called
    Derived2.VirtualMethod() called
    Base constructed
    Derived1 constructed
    Derived2 constructed
    */
    ```
    > 順序是：
    > 1. 最頂層建構子。
    > 2. 最頂層建構子中呼叫可覆寫方法時，往下找到最底層的覆寫並執行，本例中如果 `Derived2` 沒有覆寫 `AbstractMethod()` 或 `VirtualMethod()`，則會執行有覆寫的最底層 `Derived1.AbstractMethod()` 與 `Derived1.VirtualMethod()`。
    > 3. 次頂層建構子。
    > 4. 依序往下一層建構子執行。
    > 
    > 結論可擴大為"避免在任何建構子中呼叫可被覆寫的任何方法"，就算一開始正常，但只要類別被繼承而成員被覆寫，行為可能就會超出預期且很難意識到。

    但偶爾會有例外，例如在建構子中根據建構子傳入的參數初始化虛擬屬性 (Virtual Property) 就是可以接受的，但得要謹慎分析風險並在該屬性上加上相關註解。  
    > 可能是因為屬性本來就應該非常單純，不應該存在衍生類別的屬性行為和基底類別不同。  

#### 型別建構子的設計
型別建構子又稱靜態建構子 (Static Constructor)，CLR 會在型別初始化或第一個靜態類別被呼叫時去呼叫靜態建構子。

+ **X 不應** 在靜態建構子中拋出例外。
    如果拋出例外那在這個應用程式定義域 (Application Domain) 中就再也無法使用這個型別了。  
    > 用下面的範例來實驗：
    ``` csharp
    public class Cls
    {
        public static long Ticks = 0;
        static Cls()
        {
            if (Ticks == 0)
            {
                Ticks = DateTime.Now.Ticks;
            }

            Ticks.Dump();

            if (Ticks % 2 == 0)
            {
                throw new Exception();
            }
        }
    }

    try
    {
        new Cls();
    }
    catch (Exception ex)
    {
        ex.Dump();
    }

    // If an exception was previously thrown, it will be thrown again here.
    try
    {
        Cls.Ticks.Dump();
    }
    catch (Exception ex)
    {
        ex.Dump();
    }
    ```

+ **O 考慮** 直接初始化靜態欄位，而不是明確使用靜態建構子，因為當類型沒有明確定義的靜態建構子時，CLR 可以最佳化該型別的效能。
    ``` csharp
    // unoptimized code
    public class Foo
    {
        public static readonly int Value;
        static Foo()
        {
            Value = 63;
        }

        public static void Printvalue()
        {
            Console.WriteLine(Value);
        }
    }

    // optimized code
    public class Foo
    {
        public static readonly int Value = 63;
        public static void PrintValue()
        {
            Console.WriteLine(Value);
        }
    }
    ```

    CHRISTOPHER BRUMME 表示直接初始化靜態欄位會讓欄位初始化的時機難以保證。只保證欄位在被存取前會初始化，而具體可能在非常早的階段就初始化了。CLR 保留初始化的決定權，甚至在程式碼執行前就初始化了(透過 NGEN 技術)。而明確定義靜態建構子能精確的確保在靜態成員第一次被存取前執行，而不會提早執行。  

### 事件設計原則

> 事件我沒直接用過，除了畢業專題用 ASP.NET WebForm 那種自動建立的按鈕事件之類的之外 (相當於完全不懂)，所以這章內容會傾向純筆記，不會有更多深入的想法了。  
    
+ **O 應該** 使用 "raise" 而不是 "fire" 或 "trigger" 來描述事件被發起。  

+ **O 應該** 使用 `System.EventHandler<T>` 而不是建立新的委派。
    ``` csharp
    public class NotifyingContactCollection : Collection<Contact>
    {
        public event EventHandler<ContactAddedEventArgs> ContactAdded;
    }
    ```
    但如果現有專案不是這樣做，為了一致性也不用特地改掉。 這個規則也不是用於需要在不支援泛型的 CLR 版本上執行的情境。

+ **O 考慮** 除非有把握永遠不需要傳遞其他資料，否則就另外定義 `EventArgs` 的衍生類別作為參數而不要直接用 `EventArgs`。
    ``` csharp
    public class AlarmRaisedEventArgs : EventArgs
    {
    }
    ```
    如果直接使用 `EventArgs`，那在發行出去後要再為了多傳資料而擴充時就會造成破壞性變更，但如果是衍生類別的話只需要增加屬性就好了。

+ **O 應該** 將發起事件的方法設計成只有一個參數，其型別是事件參數類別 (Event Argument Class) 並將參數命名成 `e`。
    ``` csharp
    protected virtual void OnAlarmRaised(AlarmRaisedEventArgs e)
    {
        EventHandler<AlarmRaisedEventArgs> handler = AlarmRaised;
        if (handler != null)
        {
            handler(this, e);
        }
    }
    ```

+ **X 不應** 在發起非靜態 (Nonstatic) 事件時傳入 `null` 作為 sender。
+ **O 應該** 在發起靜態 (Static) 事件時傳入 `null` 作為 sender。
    > ChatGPT 說的： 兩個規則差別在於靜態事件和實例 (Instance) 無關，所以 sender 理應是 `null`，但非靜態事件和實例息息相關所以不應該傳入 `null`。  

+ **X 不應** 在發起事件時傳入 `null` 作為事件資料參數 (Event Data Parameter)。  
    應該傳入 `EventArgs.Empty`。  

+ **O 考慮** 發起使用者可以取消的事件，只適用於前置事件 (Pre-event)。  
    ``` csharp
    void ClosingHandler(object sender, CancelEventArgs e) {
        e.Cancel = true;
    }
    ```

#### 自訂事件處理常式 (Event Handler) 的設計
歸納所有規則，總之事件處理常式應該長這個形狀 `public delegate void MyEventHandler(object sender, MyEventArgs e);`，幾乎所有地方幾乎都要一模一樣，除了以下幾個地方：
+ `public` 和 `MyEventHandler` 除外。
+ 第二個參數型別通常要是 `EventArgs` 的衍生類別。

### 欄位的設計
+ **X 不應** 把欄位設成 `public` 或 `protected`。
    > 我會傾向 "只能把欄位設成 `private`"，需要對外提供操作時只能用屬性或方法。  
    > 這也是欄位和屬性在用途上最大的差異，只要欄位不是 `private` 那他就會和屬性的對外用途部分重疊並造成維護時的混亂。  

+ **O 應該** 將預設情境設計成 `static readonly` 的欄位。
    ``` csharp
    public struct Color
    {
        public static readonly Color Red = new Color(0x00FF00);
        public static readonly Color Green = new Color(0x00FF00);
        public static readonly Color Blue = new Color(0xFF0000);
    }
    ```
    > 是個對呼叫端很友善的設計，但不常用就可能忘記用。  

+ **X 不應** 將可變 (Mutable) 型別設計成唯獨欄位。
    ``` csharp
    public class SomeType
    {
        public static readonly int[] Numbers = new int[10];
    }

    SomeType.Numbers[5] = 10; // changes a value in the array
    ```
    > 屬性也一樣，如果要達到 "真唯讀" 就應該使用 `ReadOnlyList<T>`、`ReadOnlyDictionary<TKey,TValue>` 等確定內容不可變的型別。

### 擴充方法的設計

> 擴充方法設計上有一個前提是，由擴充對象提供這個方法是合理的但卻沒有提供，所以我們用這個語法糖來讓它看起來有提供這個方法，例如有一個 `string` 的擴充方法 `IsMyFriend()`，只是為了讓呼叫端的 `"John".IsMyFriend()` 呼叫比較方便，就是濫用擴充方法 (雖然這個例子很荒謬，但實務上真的有人做過類似的事)。

+ **O 考慮** 在下列任一情境使用擴充方法  
    - 當要提供方法是用於某介面下的所有實作型別時，設計該介面的擴充方法。
    - 當一個方法會破壞物件之間的依賴規則時。 例如：當 `string` 不應該和 `System.Uri` 產生耦合但又需要 `string.ToUri()` 來回傳 `System.Uri` 時就很適合設計成擴充方法來避免不適當的耦合。
    
JOE DUFFY 表示擴充方法可以用來提供介面的實作方法，例如如果已經發布了一個 `IFoo` 介面，可以提供一個 `public static void Bar(this IFoo f)` 方法讓 `IFoo` 的所有實例彷彿有 `Bar` 方法可用。  

+ **X 避免** `object` 的擴充方法。  
    有些語言例如 VB 不支援。

+ **X 不應** 將擴充方法的命名空間設計成和被擴充型別一樣，除非是為了幫介面擴充常用方法或管理依賴性。
    > 關於擴充方法的命名空間，原本我覺得它和被擴充對象有一樣的命名空間比較好，因為這樣使用者不用特別去認識擴充方法的命名空間就能直接使用。 但後來發現個致命的問題，如果有兩個以上的套件針對同一個型別設計相同的擴充方法且都用被擴充對象的命名空間時，編譯器會因為有重複的方法而編譯失敗，因此這樣設計會讓使用者在使用多個套件時有機會產生套件之間不相容的問題。  
    > 所以雖然書中是說在某些情境可考慮將擴充方法的命名空間設計成和被擴充型別一樣，但因為風險太大了所以我傾向不要這樣設計。

+ **X 不應** 在與其他功能相關的命名空間中定義實作某功能的擴充方法。相反地，應該在與該功能相關的命名空間中定義它們。 
例如，不要在 `System` 命名空間（`string` 的命名空間）中定義 `Uri.ToUri(this string string):Uri` 的擴充方法。這樣的擴充方法應該在 `System.Net` 命名空間中定義。  
+ **X 避免** 把擴充方法放在像是 `*.Extensions` 這樣的通用命名空間中，而是根據功能將其放在像是 `*.Routing` 這樣的命名空間中。  
    > 關於擴充方法命名空間，因為我之前做的都是比較輕量級的輔助套件，所以擴充方法的命名空間我是直接放在 `MyLib.Extensions`。 雖然違反了書中建議，但這有一個好處是讓使用者可以輕易的找到所有擴充方法並引入使用，如果分散在其他命名空間中一來對我來說比較難管理，另一方面使用者也不容易發覺可以用的擴充方法。  
    > 但這是因為我做的是小型套件，如果是面對大套件甚至於框架，是應該要採用書中的建議。
    
### 運算子多載的設計 (Operator Overloads)

+ **X 避免** 加入運算子多載，除非型別類似於原生型別。  
+ **O 應該** 在表示數字的型別中加入運算子多載(例如： `System.Decimal`)。
+ **X 不應** 花式設計運算子多載。
    例如：不應該把運算子用來結合兩個資料庫查詢語句，或是用 `>>` 來寫入 `Stream`。
    > 總之就是在說運算子多載應該只在適合的時候使用，基本原則就是該型別套上運算子要是合理的，然後不要把運算子拿來做過於複雜或乍看之下巧妙但很不直觀的奇怪用途。

+ **O 應該** 成對設計運算子多載。  
    例如：實作了 `==` 運算子就應該一併實作 `!=` 運算子，或是 `<` 和 `>` 也應該成對存在，依此類推。  

    RICO MARIANI 表示，如果沒有辦法實作整組的運算子多載(因為不是所有情境都合理)，那代表這時候運算子多載不是最好的設計。要記住少打幾個字沒什麼好處。

+ **O 考慮** 提供與運算子多載對應的方法。  
    因為很多語言不支援運算子多載，所以要提供 `Substract` 方法來對應 `-` 運算子。
    > 如果不打算支援多語言，我會考慮忽略這個建議。  
    > 書中有一張很大的表來列出運算子和名稱的對應，但章篇幅太長了所以需要用到時再查吧。

#### `==` 運算子
`==` 運算子多載其實很複雜，因為要相容像 `Object.Equals` 這樣的方法，更多細節在 8.9.1 章節中。

> 真的很複雜，之前做過一個 `HashName` 結構，還真沒把握 `==` 運算子實作的夠完善。  

#### 轉換運算子 (Conversion Operators)
> 就 `implicit` (明確轉換運算子) 和 `explicit` (隱含轉換運算子)

+ **X 不應** 實作使用者無法預期的轉換運算子。  
    基本上需要有調查資料佐證或是類似的現有技術有這樣使用才真的需要。  

+ **X 不應** 實作超出該型別的領域外的轉換運算子。
    `int`、`double`、`decimal` 都是數字，他們之間的轉換運算子就很合理，但是 `long` 和 `DateTime` 之間的隱含轉換就不合理，這時候應該是設計成建構子比較適合，如下：
    ``` csharp
    public struct DateTime
    {
        public DateTime(long ticks)
        {
            //...
        }
    }
    ```

+ **X 不應** 在破壞性轉換的情境實作隱含轉換運算子。
    例如： `double` 轉 `int` 可能會不正確，但明確轉換運算子仍然可以實作。
+ **X 不應** 在隱含轉換時拋出例外。
    對使用者來說出錯時很難知道發生什麼事。
    > 隱含轉換就不需要任何額外的程式就能轉型，使用者很難發現觸發了隱含轉換，所以設計像傾向絕對正確與相容；而明確轉換的情境使用者是有意識到自己正在操作轉型的，所以允許拋出例外。

+ **O 應該** 在破壞性轉型時發生但不應該容忍時拋出 `System.InvalidCastException`。  
    ``` csharp
    public static explicit operator RangedInt32(long value)
    {
        if (value < Int32.MinValue || value > Int32.MaxValue)
        {
            throw new InvalidCastException();
        }
        return new RangedInt32((int)value, Int32.MinValue, Int32.MaxValue);
    }
    ```

### 參數的設計
+ **O 應該** 使用盡可能基底的類別作為參數型別。
    能用 `IEnumerable` 就不要用 `ICollection`。
    > 這樣才能支援更多衍生型別作為引數傳入。

+ **X 不應** 在公開方法的參數中使用指標、指標陣列或是多維陣列。
    這些型別相對容易誤用。

    RICO MARIANI 表示有些人會想用這些方式來提升效能，但如果幾乎無法正確使用的話這樣的設計根本沒有幫助。  

+ **O 應該** 將所有 `out` 參數放在傳值參數與 `ref` 參數後面，除了 `params` 參數之外，且即使這會造成參數順序不一致也應如此。  
    因為 `out` 參數可視為額外的回傳值，且將他們放在一起比較好讀。
    > 我會避免使用 `out` 參數，回傳值就該是回傳值，想不到設計成 `out` 有什麼好處。

#### 選擇列舉或布林值作為參數的標準

+ **O 應該** 在成員參數有兩個參數以上的情境下使用列舉。
    不容易看出 true / false 各代表什麼參數的範例：
    ``` csharp
    Stream stream = File.Open("foo.txt", true, false);
    ```

    能輕易看出參數意義的範例：
    ``` csharp
    Stream stream = File.Open("foo.txt", CasingOptions.CaseSensitive, FileMode.Open);
    ```

    > 對我來說，我會優先用其他設計避免多個布林參數，真的不行才會選擇用列舉參數。

+ **X 不應** 使用布林，除非很確定永遠不會有第三個選項。

#### 引數的驗證 (Validating Arguments)
+ **O 應該** 要驗證列舉參數。
    列舉引數值技術上是可以超過列舉包含的範圍的。
    > 例如：`Do((OneDigit)10)`是可以編譯執行的，即使 `OneDigit` 只定義了 1 到 9。

    VANCE MORRISON 表示他並不堅信必須驗證 `[Flags]`，如果驗證了所有用不到的旗標會使得增加旗標成為破壞性變更，所以預設應該忽略沒用到的旗標。 要注意的是如果某些旗標組合是不合法的，仍然可以驗證並拋出適當的例外。

+ **X 不應** 使用 `Enum.IsDefined` 來做列舉範圍的驗證。
    BRAD ABRAMS 表示使用 `Enum.IsDefined` 有兩個缺點，第一是他會載入反射和一堆後設資料 (Loads reflection and a bunch of cold type metadata)，造成意外高昂的開銷；第二是改版時會造成問題。
    ``` csharp
    public void PickColor(Color color)
    {
        // the following check is incorrect!
        if (!Enum.IsDefined(typeof(Color), color))
        {
            throw new InvalidEnumArgumentException(...);
        }
        // issue: never pass a negative color value
        NativeMethods.SetImageColor(color, byte[] image);
    }

    // callsite
    Foo.PickColor((Color)-1); //throws InvalidEnumArgumentException
    ```
    上面的程式碼，當有人在維護時加入一個 `Ultraviolet = -1` 的項目時，這個方法就會讓 `-1` 通過驗證。

    > 不過從另一個角度來說，就算不驗證列舉範圍，預期外的值還是得有因應機制，這個因應機制同樣可能造成上述問題，所以對我來說這個規則主要還是在於效能問題上。  
    > 
    > 關於加入一個 `Ultraviolet = -1` 的項目時產生的問題，ChatGPT 有更詳細的解釋如下：  
    > Adding "Ultraviolet = -1" to the Color enum is problematic for the sample code because the check Enum.IsDefined(typeof(Color), color) is intended to ensure that only valid, predefined enum values are used. Initially, the enum might only contain positive values, so casting -1 to Color would not match any defined enum values, and the method would throw an InvalidEnumArgumentException, which is the expected behavior.  
    > However, once you add a negative value to the enum (like "Ultraviolet = -1"), the cast of -1 to Color suddenly becomes valid because it matches the newly added "Ultraviolet" value. This means that the InvalidEnumArgumentException will no longer be thrown. If the rest of the code, such as NativeMethods.SetImageColor, assumes that all valid Color values are positive, this can lead to unexpected behavior, such as a buffer overrun.  
    > A buffer overrun occurs when a program writes data beyond the end of the allocated memory buffer. This can lead to crashes, corrupted data, or even security vulnerabilities. In the context of the example, if SetImageColor is designed to handle only positive values and it receives a negative value, it could write data to an unintended location in memory, causing a buffer overrun.  
    > So, in summary, adding "Ultraviolet = -1" creates a discrepancy between what the enum check considers valid and what the rest of the application (or other applications, in the case of an API) expects as valid, leading to potential errors and security issues. This is a design consideration that must be carefully managed when working with enums and interfacing with native APIs or any other lower-level operations that may have strict assumptions about the data they receive.

+ **O 應該** 注意引數內容在驗證後仍然可能被改變。
    如果對於安全性很在意，應該要完整複製內容後再驗證與操作該引數。

    > 這個很難理解，大概用下面的程式碼說明：
    ``` csharp
    public void ProcessData(Buffer data)
    {
        if (!data.IsValid())
        {
            throw new SecurityException("Data is not valid.");
        }
        
        InternalProcess(data);
    }
    private void InternalProcess(Buffer data)
    {
        // Implementation for processing the data.
    }
    ```
    > `Buffer` 是參考型別且**呼叫端可能跨執行序改變他的內容**，在執行 `InternalProcess(data)` 的期間 `data` 的值可能被其他執行序改變。  
    > 這時候就應該改用完整複製後的內容來驗證並執行後續行為如下：  
    ``` csharp
    public void ProcessData(Buffer data)
    {
        // Make a copy of the data for validation and processing.
        Buffer dataCopy = data.Clone();

        // Validate the copied data.
        if (!dataCopy.IsValid())
        {
            throw new SecurityException("Data is not valid.");
        }

        // Since we have validated the copy and we know it hasn't changed since validation,
        // we can now safely process this copy.
        InternalProcess(dataCopy);
    }

    private void InternalProcess(Buffer data)
    {
        // Implementation for processing the data.
    }
    ```
    >
    > 老實說我認為這個建議所提到的問題過於悲觀了，如果所有參考型別的引數都這樣做的話成本會嚇死人。只能做為一個考量並只在非常必需時使用。

#### 參數傳遞

+ **X 避免** 使用 out 或 ref 參數。  
    使用這兩種參數要對他們背後的機制了解的很透徹，對使用者的技術要求太高。  

+ **X 不應** 用傳址方式傳遞參考型別。

#### 擁有可變參數數量的成員
> 就是在說 `params` 關鍵字。

+ **O 考慮** 在呼叫端可能傳入少量參數且適合使用 `params` 時使用。  
    如果都是傳入大型陣列，那呼叫端會整理好一個陣列後傳入，此時 `params` 是非必要的。  
+ **X 避免** 在呼叫端都傳入陣列的情境使用 `params`。  
    例如 `byte[]` 為參數的情境，幾乎不可能使用到 `params` 的特性。

+ **O 考慮** 在簡單的多載使用 `params` 即使其他多載可能不用。
    ``` csharp
    public class Graphics
    {
        FillPolygon(Brush brush, params Point[] points) { }
        FillPolygon(Brush brush, Point[] points, FillMode fillMode) { }
    }
    ```
    如果第一個多載很常用，這樣設計是可以的，不必受第二個多載的樣式限制。

+ **O 應該** 透過參數順序讓 `params` 可用。

+ **O 考慮** 對於需要極致效能的 API，提供包含少數參數的特殊多載來取代 `params`。
    ``` csharp
    void Format (string formatString, object arg1);
    void Format (string formatString, object arg1, object arg2);
    void Format (string formatString, params object[] args);
    ```
    如上，呼叫第二個多載比呼叫第三個多載少建立一個臨時的陣列物件。
    > 這個太特殊了通常用不到，但是 "多建立一個臨時的陣列物件" 的是非常細膩的考量所以還是記一下。

#### 指標參數
因為指標不符合 CLS 規範 (Not CLS-compliant)，所以通常指標不應該用於公開 API，但在需要使用的時候...

+ **O 應該** 提供另外一個替代方案來搭配包含指標參數的方法。
    ``` csharp
    [CLSCompliant(false)]
    public unsafe int GetBytes(char* chars, int charCount, byte* bytes, int byteCount);

    public int GetBytes(char[] chars, int charIndex, int charCount, byte[] bytes, int byteIndex, int byteCount);
    ```

+ **X 避免** 花太多成本驗證指標參數。
    通常驗證參數的開銷是必要的，但是會用到指標參數代表很在意效能，這個脈絡下驗證參數的成本就顯得太高。

+ **O 應該** 遵循指標相關設計慣例。
    例如，下方的程式碼來說，起始索引是不必要的，因為呼叫端能用其他方式達到一樣的效果。  
    ``` csharp
    // Bad practice
    public unsafe int GetBytes(char* chars, int charIndex, int charCount, byte* bytes, int byteIndex, int byteCount);

    // Better practice
    public unsafe int GetBytes(char* chars, int charCount, byte* bytes, int byteCount);

    // Example call site
    GetBytes(chars + charIndex, charCount, bytes + byteIndex, byteCount);
    ```

### 結論
這章本來就很大篇幅，又因為細節很多，很多看似理所當然的內容卻容易在開發時忽略，所以沒太多可以省略的之外還多了一堆實驗範例和註記而加長篇幅。

### 參考
[Exceptions in managed threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/exceptions-in-managed-threads)  

[How finalization works](https://learn.microsoft.com/en-us/dotnet/api/system.object.finalize#how-finalization-works)   

ChatGPT  