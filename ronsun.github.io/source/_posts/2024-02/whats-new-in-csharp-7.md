---
title: C# 語言特性更新 - C# 7
date: 2024-02-03 01:22:57
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

C# 7 系列比較特殊，總共經過 7.0、7.1、7.2、7.3 四個版本，且範圍有高度重疊，所以這邊不分開看，直接依照 2021 年版官方整合過的文件將 C# 7 的新特性一起看。

<!--more-->

### Tuples 和 Discards

#### 具名 Tuple
以前的 Tuple 沒辦法具名，所以會產生很多 `tuple.Item1` 這類的呼叫，嚴重影響程式碼的可讀性，具名 Tuple 出現後很大程度解決了這個問題，如下範例：
``` csharp
public (int Id, string Name) GetPerson()
{
    return (1, "Ron");
}

// Caller
var user = GetPerson();
$"ID: {user.Id}, Name: {user.Name}".Dump(); // ID: 1, Name: Ron
```

但這也衍生出另外一個問題就是命名風格，這邊我建議依照呼叫端的使用方式來決定，以本例來說，呼叫端使用上像是將他視為屬性，因此使用雙駝峰。但如果像下面的情境，就應該使用單駝峰，因為呼叫端使用上像是區域變數：
``` csharp
(var id, var name) = (1, "Ron");
$"ID: {id}, Name: {name}".Dump(); // ID: 1, Name: Ron
```

另外還要注意這種風格的 Tuple 其實背後是編譯成 `System.ValueTuple` 而不是傳統的 `System.Tuple`，[這兩種型別有一些差異](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-tuples#tuples-vs-systemtuple)。  

#### Discards

``` csharp
public (int Id, string Name, string Address) GetPerson()
{
    return (1, "Ron", "My Home");
}

// Caller, discard the Address
(var id, var name, _) = GetPerson();
$"ID: {id}, Name: {name}".Dump(); // ID: 1, Name: Ron
```

如上範例，用不到的變數可以用一個底線 `_` 取代，可以讓維護的人不用關注無用的資訊。但這也不是他唯一的使用情境，詳細情境參考[ Discards 官方文件](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/discards)。

#### C# 怎麼處理 Tuple 的名字?
將一段簡單的程式碼編譯後，透過反組譯工具反組譯回低版本的 C# 可以發現，最後還是呼叫 `Item1`、`Item2`，所以具名 Tuple 其實是一種編譯時期的語法糖。

#### 解構 (Deconstruction)
雖然解構看起來是另外一個主題了，但可能是因為 Tuple、Discards、解構，三者太常互相搭配使用，官方文件中是將他和 Discards 放在一起的，解構不只能套用到 Tuple 還能透過實作 `Deconstruct` 方法套用到其他型別上。  

直接看 [Discards](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/discards) 和 [Deconstructing tuples and other types](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/deconstruct) 來了解會更全面。  

這邊要注意一下，寫這篇文章時 C# 已經更新到 12 版了，官方的規格介紹中會夾很多後面版本新增的特性。  

### 模式比對 (Pattern matching) 

這個功能比較像是為了可讀性的擴充，主要用途在於用 `is` 關鍵字來做型別確認與轉換；用 `when` 關鍵字來做附加條件判斷；額外搭配 `switch` 來透過簡單的語法達到複雜的運用。

#### `is` 關鍵字
``` csharp
public void Do(object obj)
{
    if (obj is int i)
    {
        // Do somthing using i.
        return;
    }

    if (obj is string s)
    {
        // Do somthing using s.
        return;
    }
}
```

從上述範例可以看到以前需要先透過 `is` 判斷型別後再轉型 (或使用 `as` 後再判斷空值後處理)，現在可以將兩個行為合一讓可維護性提高很多。

#### `switch`
``` csharp
private static void Do(object obj)
{
    switch (obj)
    {
        case int i:
            // Do somthing using i.
            break;
        case string s:
            // Do somthing using s.
            break;
        case DateTime dt when dt.Year == 2021:
            // Do somthing using dt which contains Year is 2021.
            break;
        case null:
            // Do somthing for null scenario.
            break;
        default:
            // Default behavior.
            break;
    }
}
```

從上述範例可以看到，`switch` 關鍵字在使用上能更有彈性的設定條件了。  

這個規格依然是編譯時期的語法糖， `switch` 本身的限制並沒有改變，上面的範例編譯後其實是 `if`、`else if`、`else` 條件式的組合。

### Async main
在 Main 上可以加上 `async` 關鍵字了。  
比較意外的是這也是語法糖，[保哥有一篇文章提供詳細的說明](https://blog.miniasp.com/post/2019/04/03/Deep-Dive-CSharp-71-async-Main-method)。

### 區域函式 (Local functions)
區域函式允許一個方法中包含另外一個方法，而區域函式只能被包含他的方法呼叫。這個設計主要用於迭代器和非同步方法中。  

#### 實務使用情境
而在實務上，在滿足以下條件下，也可考慮使用區域函式：
+ **有足夠的理由將該區塊抽離：** 通常這種情境下如果不抽離的話，原方法會過於肥大，以前常見的作法是用 `#region` 包覆起來，但這種做法對維護沒有太大的幫助。
+ **該方法做為私有方法仍然過於特殊：** 很多時候會有程式碼區塊內容過於特殊，離開主方法後無法封裝成一個方法。  

#### 使用注意事項
區域函式不宜濫用，如果濫用會發現主要方法仍然很肥大，且會增加排版的難度。關於區域函式排版上我自己有幾個建議要點：
1. 區域函式集中放在最下方且要在 `return` 後，這樣區域函式才不會穿插在呼叫端的程式碼中造成更大的混亂。
2. 即使方法無回傳也要在區域函式前加上 `return;`，用以更明確的區隔呼叫端程式碼和區域函式部分。

如下範例：
``` csharp
public void Do(int a, int b, bool c)
{
    var a2 = LocalFunction(a);
    int ans = c ? LocalFunction(a2 + b) : LocalFunction(a * b);

    // Always return at the last line of primary implementation.
    return;

    // Local functions
    int LocalFunction(int num)
    {
        return num * 2;
    }
}
```

區域函式也是編譯時期的語法糖且有很多細節要說，尤其是參數傳遞的部分，這部分之後再另外寫一篇來詳細說明。  

### 更多成員支援運算式主體 (More expression-bodied members)
運算式主體套用到更多成員上，包含建構子、Finalizer、以及在屬性和索引子上的 `get` 與 `set` 存取子，範例如下：

``` csharp
public class MyContainer
{
    private List<string> items = new List<string>();
    private string label;

    // Expression-bodied constructor
    public MyContainer(string initialLabel) => Label = initialLabel;

    // Expression-bodied finalizer
    ~MyContainer() => Console.WriteLine("Finalizing or cleaning up MyContainer.");

    // Expression-bodied property get and set accessors
    public string Label
    {
        get => label;
        set => label = value ?? "Default label";
    }

    // Expression-bodied indexer
    public string this[int index]
    {
        get => items[index];
        set => items[index] = value;
    }
}
```

要注意的是 [Finalizer，舊稱 Destructor](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/finalizers) 的語法 `~` 實際上是個語法糖，編譯後其實是 `Finalize()` 方法。


### throw 運算式 (Throw expressions)
`throw` 一直都是陳述式 (Statement) 而非運算式 (Expression)，也因此以前很多運算式中無法包含 `throw` (例如在三元運算子中)，而在 C# 7.0 中新增的 [throw 運算式](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/exception-handling-statements#the-throw-expression) 解決了這個限制。  

### 預設常值運算式 (Default literal expressions)
[擴大 `default` 運算子的適用範圍到運算式上](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/default#default-literal)  

### 數值常值的語法增強功能 (Numeric literal syntax improvements)  
允許利用 `_` 符號來分隔數字而不影響數值本身，類似於千分位符號的功能且同時能套用到二進位數值上，範例如下：
``` csharp
int binary = 0b001_0000;
int number = 10_000_000;
decimal money = 1_000.123_456;
```

### `out` 變數 (`out` variables)
可以把 `out` 變數的宣告和傳遞寫在一起，如下：
``` csharp
int.TryParse(str, out int number);
```

這個特性雖然看起來不起眼，但是在實務上對於維護是有非常大的幫助的，他能避免讓變數的宣告和使用的位置距離太遠，如下面範例是實務上常見的痛點。  
首先看這段程式碼，因為維護或各種關係導致變數 `number` 和 `decimal` 的宣告遠離第一次被使用的位置，進而造成閱讀時必須同時關注這麼一大段程式碼並忽略中間穿插的各種雜訊。
``` csharp
int number;

// Unrelated code, making it hard to track 'number'
for (int i = 0; i < 10; i++) {
    Console.WriteLine("Loop 1: " + i);
}
string temp = "Hello";
Console.WriteLine(temp.ToUpper());
bool flag = true;
if (flag) {
    Console.WriteLine("Flag is true");
}

decimal money;

// Unrelated code, making it hard to track 'money'
foreach (var item in new List<string> { "a", "b", "c" }) {
    Console.WriteLine("Item: " + item);
}
DateTime now = DateTime.Now;
Console.WriteLine("Current time: " + now);

// Usage of 'number' and 'money' is separated from its declaration
int.TryParse(str, out number);
decimal.TryParse(str2, out money);
```

但套用這個特性，維護過程我們就不用關注那些雜訊而能直接鎖定在真正的目標上。
``` csharp
int.TryParse(str, out int number);
decimal.TryParse(str2, out decimal money);
```

這個問題的主因是不好的開發習慣，但長期多人開發的專案很難完全避免這個現象。重構和 Code Reivew 能緩解這個問題，但終究是額外的成本，因此從語言特性上直接避免是更理想的。


### 非後置具名引數 (Non-trailing named arguments)
具名引數非必要放在最後，例如：
``` csharp
// In C# 7.1 and earlier, named arguments should be trailing.
DisplayPersonDetails(35, "Ron", city: "New York");
// In C# 7.2 and later, named arguments do not necessarily have to be trailing.
DisplayPersonDetails(age: 35, "Ron", city: "New York");
```

這讓我們可以不用為了遷就前排的具名引數而將其後所有引數具名，但從另外一個角度來看，使用具名引數時提高可讀性前應該先思考方法參數列的設計是否需要改善。  

### `private protected` 存取修飾詞 (`private protected` access modifier)
限制存取範圍在 **`protected` AND `internal`**，比 `protected internal` 代表的 **`protected` OR `internal`** 存取範圍更狹窄。  

### 改善選擇多載的規則 (Improved overload candidates)
增加三個解析規則來讓更明確的選擇多載。  

老實說我不想特別去記這些複雜的規則，因為這些規則是建立在呼叫多載方法時會混淆的前提，通常這代表多載設計得不夠好，應該優先考慮改善多載方法的設計。

### 擴充安全程式碼的能力 (Enabling more efficient safe code)
幾乎都是沒用過的特性，所以直接看官方文件，沒甚麼好說的。

### 套用 `ref` 到區域變數和回傳值上 / `ref` 條件運算式 (Ref locals and returns / Conditional `ref` expressions)
允許傳址給區域變數，例如：  
``` csharp
public class Program
{
    public static void Main()
    {
        int[] numbers = { 1, 2, 3 };
        ref int two = ref numbers[1];
        
        two = 0;

        Console.WriteLine(two); // 0
        Console.WriteLine(numbers[1]); // 0
    }
}
```

允許回傳時傳址，例如：  
``` csharp
public class Program
{
    public static void Main()
    {
        int[] numbers = { 1, 2, 3 };
        ref var two = ref GetTwo(numbers);

        two = 0;

        Console.WriteLine(two); // 0
        Console.WriteLine(numbers[1]); // 0
    }

    static ref int GetTwo(int[] numbers)
    {
        return ref numbers[1];
    }
}
```

`ref` 條件運算式，例如：
``` csharp
ref var i = ref (numbers1 != null ? ref numbers1[0] : ref numbers2[0]);
```

除非很必要否則我不用 `ref` 的，因為對於開發、維護人員來說門檻較高且容易被濫用，且到目前為止工作上可以使用 `ref` 的情境其實都有其他成本不高且低副作用的替代方案。 當然也不是說 `ref` 不好，相信他在一些效能極度敏感的情境是很有用的。  

### `in` 參數修飾詞
補充現有的 `ref` 和 `out`。  
`ref` 傳址，被呼叫的方法不一定要為其賦值；`out` 傳址，被呼叫的方法一定要為其賦值，代表這個引數在傳入前賦值通常是沒意義的；`in` 傳址，被呼叫的方法不能為其賦值。  
``` csharp
void Do(ref string a, out string b, in string c)
{
    // Must to have
    b = "B";
    // Can not assign to "in" parameter.
    // c = "C";
}

string s1 = "a";
// Bad, assignment here generally doesn't make sense.
string s2 = "b";
// Need to have value generally.
string s3 = "c";
Do(ref s1, out s2, in s3);
```

### 更多型別支援 `fixed` 陳述式 (More types support the `fixed` statement)
在 C# 7.3 之後，所有包含 `GetPinnableReference()` 方法且回傳 `ref T` 或 `ref readonly T` 的型別都能套用 `fixed`。

### `fixed` 索引欄位不需要釘選 (Indexing `fixed` fields does not require pinning)
``` csharp
using System;

public unsafe struct MyStruct
{
    public fixed int FixedArray[10];
}

public class Example
{
    public static void Main()
    {
        MyStruct myStruct = new MyStruct();

        // Before C# 7, pinning required
        fixed (int* ptr = myStruct.FixedArray)
        {
            for (int i = 0; i < 10; i++)
            {
                ptr[i] = i;
            }
        }

        // After C# 7, no pinning required
        for (int i = 0; i < 10; i++)
        {
            // Direct indexing is now allowed
            myStruct.FixedArray[i] = i;
        }
    }
}
```

### `stackalloc` 陣列支援初始設定式 (`stackalloc` arrays support initializers)
C# 7.3 之前：
``` csharp
int* arr = stackalloc int[3];

arr[0] = 1;
arr[1] = 3;
arr[2] = 5;
```

C# 7.3 之後：
```csharp
int* arr = stackalloc int[3] { 1, 3, 5 };
```

### 擴充泛型限制
可以限制 `System.Enum` 或 `System.Delegate` 做為泛型型別的限制；也可以用 `unmanaged` 限制為不可空的非託管型別。  

最有感的是 `System.Enum` 以前要限制泛型型別只能用 `struct` 再加上一些驗證程式碼，沒辦法在編譯時期精準限制列舉為泛型型別。

### 新的編譯選項
這部分沒用過，就[留個連結備查](https://web.archive.org/web/20211114194546/https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-7#new-compiler-options)。

### 結論
C# 7 多了很多東西，最明顯的是各種語法糖，面對語法糖，應該找時間更仔細的研究他們編譯後實際的樣子避免誤用。

### 參考
[What's new in C# 7.0 through C# 7.3 (原文件已經被刪除，這是 Archive 網站的存檔)](https://web.archive.org/web/20211114194546/https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-7)