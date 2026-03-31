---
title: Framework Design Guidelines 整理與心得 (7)
categories:
  - Reading
  - Framework Design Guidelines
tags:
---

這系列文章的目的是為了之後參考與快速回顧而基於 Framework Design Guidelines 這本書寫的整理與心得，這不是翻譯，所以內容都是閱讀理解後再梳理下來的，和書中語句必然不同，也只會保留我自己覺得需要的部分，並帶一些自己的想法與註解。

本篇包含了第七章的部分。 

<!--more-->

例外永遠只用來回報例外，不能有其他用途。

### 例外拋出 (Exception Throwing) 的設計

+ **O 應該** 在出錯時拋出例外。
+ **X 不應** 在出錯時回傳錯誤狀態。
    > 除非部分成功，也就是批次或多步驟操作，允許部分成功且回報執行細節的場景。

JASON CLARK 表示當方法無法達到定義的行為時就應該拋例外，例如：ReadByte 應該在 stream 沒有可讀位元組時拋例外，但 ReadChar 不應該在 stream 結束拋例外，因為 EOF 通常是合法字元，所以在 stream 結尾可以讀到 char 但不能讀到 byte。  
> 這個可以用在方法的邊界情境設計時，思考是否要支援空值、參數空集合、是否回傳 null 等。

+ **O 考慮** 在極端現象以至於系統無法安全繼續運作時，直接呼叫 `System.Environment.FailFast` 終止取代拋出例外。  

+ **X 不應** 用例外做為一般流程控制用途。

+ **O 應該** 將所有公開成員可能拋出的例外寫入文件，因為這是規格的一環。改版時不應該更改例外型別，也不應該增加新的例外。
    > C# 的 [XML API documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/) 就包含這部分。  
    > 我的理解是，改版時若變更既有例外契約，應視為破壞性變更。

KRZYSZTOF CWALINA 表示如果改變拋出類別，但拋出的類別是原本的例外的子類別的話，不算破壞性變更且可考慮，因此在需要新的例外型別時，若非必要則優先考慮繼承自原本的型別而不是建立不相干的型別。  

+ **X 不應** 透過參數（或其他方式）決定是否拋出例外。
    ``` csharp
    // bad design
    public Type GetType(string name, bool throwOnError)
    ```

+ **X 不應** 回傳或透過 `out` 輸出例外。
    ``` csharp
    // bad design
    public Exception DoSomething() {... }
    ```

+ **X 不應** 在 Exception Filter 中拋出例外。
    ``` csharp
    try
    {
        DoSomething();
    }
    // when 區塊有例外不會捕捉到
    catch (Exception ex) when (ex.Message.Contains("timeout"))
    {
        Console.WriteLine("Timeout occurred");
    }
    ```

+ **X 避免** 在 `finally` 區塊中主動拋出例外，但如果是呼叫其他方法而那個方法可能會拋出例外則可以接受。

### 拋出正確的例外型別

+ **O 考慮** 在需要特別辨識的情況下拋出客製化的例外型別，否則就拋出預設的就好。

+ **O 應該** 拋出合理範圍內最具體 (繼承階層中最底層的衍生類別) 的例外。

#### 錯誤訊息

+ **O 應該** 提供詳盡的訊息，包含說明錯誤內容與建議的因應作為。
    > 其他就是一些常識，文法正確、句末要有句點、不要放問號或驚嘆號、不要放敏感資訊、考慮多國語系等。

#### 例外處理
+ **X 不應** 在框架中透過捕捉過於泛用的例外例如 `System.Exception` 來忽略錯誤。
+ **X 避免** 在應用程式中透過捕捉過於泛用的例外例如 `System.Exception` 來忽略錯誤。
    非常罕見的情形下可能會需要在應用程式中這樣做，但要確定是深思熟慮後的決定。

#### 包裝 (Wrapping) 例外
+ **O 考慮** 當低層例外對高層 API 使用者沒有意義時，重新包裝例外。
    ``` csharp
    try
    {
        // read the transaction file
    }
    catch (FileNotFoundException e)
    {
        throw new TransactionFileMissingException(..., e);
    }
    ```
    > 從應用程式的角度來看，實務上通常都是順著 stack trace 往下追問題點，所以刻意再包一層常常顯得很多餘。  
    > 但站在框架設計的角度，得考慮不同使用者的偵錯習慣，因此例外型別與訊息往往還是要更細緻。

ERIC GUNNERSON 表示不要讓使用者需要去看 inner exception 才能正確使用你的 API。

> 整體來說，即使需要重新包裝，包裝後的例外訊息也要有足夠的描述性，而不是讓使用者還得再深入 inner exception 才能了解問題。

### 使用標準的例外型別
這裡只做概述，細節請看 .NET 文件。  

#### `System.Exception` 或 `System.SystemException`
> 除了全域例外處理外，不應拋出或捕捉。

#### `ApplicationException`
+ **X 不應** 拋出或繼承 `System.ApplicationException`。

JEFFREY RICHTER 提過這個例外最初的設計意圖，但由於後來沒有被一致遵守，所以它已經失去原本的分類意義。

> 也就是說，官方後續沒有一致遵守這個設計，導致它失去原本的價值。

#### InvalidOperationException
+ **O 應該** 發生在物件當下狀態不允許的操作時，拋出 `System.InvalidOperationException`。
    > 原則上我會避免將物件設計成會受狀態影響可呼叫的屬性或方法。

KRZYSZTOF CWALINA 表示 `InvalidOperationException` 和 `ArgumentException` 的差別在 `ArgumentException` 是單純參數錯誤，不受其他物件的影響。  

#### `ArgumentException`、`ArgumentNullException`、`ArgumentOutOfRangeException`
+ **O 應該** 在從 setter 中拋出這類例外時，用 "value" 當參數名稱。
    > 我的理解是，這裡真正出錯的是 setter 的隱含參數 `value`，而不是屬性名稱本身。
    ``` csharp
    public string Name
    {
        set
        {
            if (value == null)
            {
                throw new ArgumentNullException("value", ...);
            }
        }
    }
    ```

#### `NullReferenceException`、`IndexOutOfRangeException`、`AccessViolationException`
+ **X 不應** 讓公開 API 拋出這類例外。
    應該做好參數驗證來避免拋出這些例外，之所以要避免是因為拋出這些例外會暴露過多的實作細節。
    > 如果這些實作細節之後改變，拋出的例外可能也改變了，這時呼叫端如果有攔截這些例外的話可能會出錯。  
    > 分別從框架設計這種不定使用者的場景和應用程式開發兩個不同視角來看，這個規則的重要度會差很多。  

#### `StackOverflowException`
+ **X 不應** 拋出這個例外，這只能由 CLR 拋出。
+ **X 不應** 捕捉這個例外。
    > 從 CLR 拋出的 `StackOverflowException` 時會直接終止流程，沒機會被捕捉到。

#### `OutOfMemoryException`
+ **X 不應** 拋出這個例外，這只能由 CLR 拋出。

CHRISTOPHER BRUMME 表示可以將這個例外當一般例外捕捉或處理，因為 CLR 無法判定是資源枯竭還是仍有餘裕所以如果不是資源枯竭那這個處理有可能會成功。  

#### `COMException`、`SEHException`、`ExecutionEngineException`
+ **X 不應** 拋出這個例外，這只能由 CLR 拋出。

CHRISTOPHER BRUMME 表示可以將這個例外當一般例外捕捉或處理。


### 客製化例外的設計
+ **O 應該** 以後綴 `Exception` 命名。

+ **O 應該** 在執行階段可序列化。

+ **O 應該** 提供固定的幾個建構子，且確保參數名字與型別完全符合下面範例。
    ``` csharp
    public class SomeException : Exception, ISerializable
    {
        public SomeException();
        public SomeException(string message);
        public SomeException(string message, Exception inner);
        // this constructor is needed for serialization.
        protected SomeException(SerializationInfo info, StreamingContext
        context);
    }
    ```

+ **O 應該** 在透過 `ToString()` 回傳敏感資訊前先驗證過權限，無權限則應該回傳篩選過的資訊。

RICO MARIANI 表示不要把 `ToString()` 的結果存成能任意存取的資料結構，除非能確保內容無法被不受信任的程式碼存取，因為裡面通常包含敏感資訊。
> 因為本書是針對框架設計，所以這裡的使用者通常是指框架使用者，也會將他們視為不可信或無法預期的對象，和一般面對上層應用程式時的認知不同。

+ **O 應該** 把有用的敏感資訊存成私有狀態，避免讓不受信任的程式碼直接存取。
> 例如存成 private member 只能從該類別中存取而不對外暴露的意思。

+ **O 考慮** 提供結構化的屬性讓程式可以讀取例外資訊，而不是只靠字串。
> Exception 型別中應該包含一些屬性可以讓呼叫端拿到關鍵脈絡資訊，如果全部放字串就會難以透過程式判定問題細節。

### 例外與效能
頻繁拋出例外會對效能造成明顯的負面影響。

#### Tester-Doer 模式
先判斷再操作避免例外，例如
``` csharp
IList<string> stringList = ...;
if (stringList != null && stringList.Count > 0)
{
    stringList[0].Dump();
}
```

#### Try-Parse 模式

這個模式對 "Try" 所涵蓋的失敗範圍應該有嚴格限制，如果使用者超出這個範圍，仍然可能會拋出例外。  
> 參考 .NET 的 TryParse 方法或 TryXXX 方法的原始碼，可以發現這個模式其實是涵蓋常見轉換錯誤的驗證，**絕對不是** 把 try-catch 封裝起來。

+ **O 應該** 以 Try 前綴命名這類方法，並回傳 `bool` 表示是否符合規範。
    > 且會有一個 `out` 參數，表示輸出結果。  
+ **O 應該** 同時提供另外一個拋例外的方法。
    ``` csharp
    public struct DateTime
    {
        public static DateTime Parse(string dateTime) { ... }
        public static bool TryParse(string dateTime, out DateTime result) { ...}
    }
    ```

### 參考
ChatGPT
