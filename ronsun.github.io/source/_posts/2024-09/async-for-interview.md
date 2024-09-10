---
title: 從面試角度看 async
date: 2024-09-10 23:38:38
categories:
- C#
- Language Spec
tags:
---

C# 的非同步程式設計是一個相當龐大且複雜的主題，甚至可以寫成整本專書。由於這也是面試中的常見題目，本文將嘗試從面試的角度整理非同步開發的要點。

<!--more-->

### 非同步程式設計的特點
在同步程式設計中，任務依序執行，程式必須等待當前操作完成後才能繼續下一個操作。例如，當程式需要讀取檔案或從網路取得資料時，必須等這些 I/O 操作完成後才能繼續執行其他操作。因此，若遇到高耗時的操作(如 I/O 或長時間的計算)，主執行緒將會被阻塞，導致程式在等待期間無法進行其他操作。  

相對地，在非同步程式設計中，當執行耗時較長的 I/O 操作時，非同步程式可以啟動該操作並立即返回，允許其他任務繼續執行。這讓 CPU 在等待 I/O 操作完成的同時處理其他操作，使系統資源得以更有效率地運用。這種特性在處理網路請求、檔案 I/O 或資料庫存取等需要等待結果的操作時特別有用。  

### `async` 與 `await` 關鍵字如何用於非同步程式
讓我們從一個範例開始：
``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

static async void Main(string[] args)
{
    // * Step 1
    var demo = new AsyncDemo();

    // * Step 2
    var finalResult = await demo.Service();

    // Step 15
    Console.WriteLine(finalResult);
}

public class AsyncDemo
{
    public async Task<string> Service()
    {
        // * Step 3
        var readerTask = ReadFromFile();

        // * Step 5
        var aggregated = Calculate()
            .Select(r =>
            {
                // Story I:
                // * Step 6
                // * Step 7
                // * Step A with another thread (T2) because Step 4 completed.
                // * Step 8 with current thread (T1).
                // * Step 9
                // * Step 10
                // ==================
                // Story II:
                // * Step 6
                // * Step 7
                // * Step 8
                // * Step 9
                // * Step 10
                Console.WriteLine($"[T{Environment.CurrentManagedThreadId}] Calculated answer: {r}");
                return $"{r}";
            })
            .Aggregate((l, r) => $"{l}, {r}");

        // * Step 11
        Console.WriteLine($"[T{Environment.CurrentManagedThreadId}] Aggregated of calculated answer: {aggregated}");

        // Story I:
        // * Step 12
        // ==================
        // Story II:
        // * Step 4 took too much time.
        // * Wait at Step 12 util Step 4 completed.
        // * Step A with another thread (T2) once Step 4 completed.
        // * Step 12 from another sequence.
        var fileData = await readerTask;

        // * Step 13
        Console.WriteLine($"[T{Environment.CurrentManagedThreadId}] File data returned.");

        // Step 14
        return $"[T{Environment.CurrentManagedThreadId}] FileData: {fileData}; Calcualted: {aggregated}";
    }

    private async Task<string> ReadFromFile()
    {
        // Step 4
        await Task.Delay(500);
        var result = "File Content: Data from file.";

        // Step A
        Console.WriteLine($"[T{Environment.CurrentManagedThreadId}] {result}.");

        var ans = Calculate()
            .Select(r =>
            {
                // Step B
                // Step C
                // Step D
                // Step E
                // Step F
                Console.WriteLine($"[T{Environment.CurrentManagedThreadId}] Calculated answer in {nameof(ReadFromFile)}: {r}");
                return $"{r}";
            })
            .Aggregate((l, r) => $"{l}, {r}");

        // Step G
        return result;
    }

    private IEnumerable<int> Calculate()
    {
        int limit = 100000000;
        int i = 0;
        while (i++ < limit)
        {
            if (i % (limit / 5) == 0)
            {
                yield return i;
            }
        }
    }
}
```

這個範例看起來較為複雜，但只要掌握幾個基本概念，我們就可以逐步理解：
1. `async` 有傳染性，所有非同步方法的呼叫端都必須是非同步方法(設計上必須，否則就要使用同步等待，這會造成很大的風險後面會說明)。
2. 當一個方法呼叫非同步方法時，非同步方法會返回一個 `Task` 或 `Task<T>`。如果呼叫方使用了 `await`，它會暫停該方法的執行，並將控制權返回給更上層的呼叫端。如果更上層的呼叫端也使用了 `await`，這個過程會繼續向上傳遞，直到找到一個不再使用 `await` 的呼叫端。此時，程式才會從那個不使用 `await` 的呼叫端繼續執行，並等到非同步操作完成後再依序往下執行。
   > 換句話說，當某個方法 `M3` 呼叫非同步方法 `M4` 時，`M4` 會回傳一個 `Task` 或 `Task<T>`。如果 `M3` 使用了 `await` 來等待 `M4` 的完成，`M3` 的執行會暫時中斷，並將控制權返回給更上層的呼叫端 `M2`。如果 `M2` 也使用了 `await`，這個過程會持續向上傳遞，直到找到一個不再使用 `await` 的呼叫端(例如 `M1`)。此時，控制權會停留在這個不使用 `await` 的呼叫端(例如 `M1`)，直到非同步操作完成，然後程式流程再依序從等待點繼續執行。

接下來我們逐步說明上面範例的執行順序：
1. 從 Step 1 開始。
2. Step 2 呼叫非同步方法 `Service()`，並使用 `await` 等待結果。
3. 進入後 Step 3 接著呼叫下一層的非同步方法。
4. Step 4 `await Task.Delay(...);` 模擬一個耗時的 I/O 操作，因為有使用 `await` 所以在操作完成前將**控制權返回給上一層的呼叫端**。
5. 回到上一層看到 Step 3 的部分沒有使用 `await`，由此繼續往下到 Step 5。
6. Step 5 ~ 10 執行了一連串 CPU Bound 的操作，視 Step 4 的 I/O 操作與這一連串 CPU Bound 操作誰先完成，會產生出兩種不同的執行順序。
   1. 第一種順序是，上一步的 CPU Bound 操作未完成但 Step 4 的 I/O 操作已完成，控制權回到 Step 4 結束後繼續往下，於此同時，Step 5 ~ 10 仍然由另外由不同的執行緒服務，Step A ~ G 會和 Step 5 ~ 11 中未完成部分交錯完成。
   2. 第二種順序是，上一步的 CPU Bound 操作先完成並往下到 Step 11。
7. 從 Step 12 開始，由於這邊有使用 `await`，**控制權返回給更上一層的呼叫端**來到 Step 2，而 Step 2 也有使用 `await`，理應將**控制權返回給更上一層的呼叫端**，但由於已經到達程式入口點，所以這邊無法繼續追蹤。
   1. 等到 Step G 完成回傳後，Step 12 等待結束繼續往下。
   2. 接著 Step 4 完成，等待結束依序往下執行 Step A ~ G 後回傳到 Step 12，接著 Step 12 等待結束繼續往下。
8. 接著是 Step 13 ~ 14 完成後回傳。
9.  Step 2 等待結束繼續往下執行 Step 15 後結束。

接著附上兩種不同執行順序的實際 Log 協助讀者理解。  
首先是第一種順序，I/O 操作結束後的程式，會和呼叫端在 I/O 操作未完成時可以同時執行的操作交錯由不同執行緒服務。
```
[T1] Calculated answer: 20000000
[T1] Calculated answer: 40000000
[T17] File Content: Data from file..
[T1] Calculated answer: 60000000
[T17] Calculated answer in ReadFromFile: 20000000
[T1] Calculated answer: 80000000
[T17] Calculated answer in ReadFromFile: 40000000
[T1] Calculated answer: 100000000
[T1] Aggregated of calculated answer: 20000000, 40000000, 60000000, 80000000, 100000000
[T17] Calculated answer in ReadFromFile: 60000000
[T17] Calculated answer in ReadFromFile: 80000000
[T17] Calculated answer in ReadFromFile: 100000000
[T17] File data returned.
[T17] FileData: File Content: Data from file.; Calcualted: 20000000, 40000000, 60000000, 80000000, 100000000
```

接著是第二種順序，I/O 操作結束前，能同時執行的操作早已全部完成。
```
[T1] Calculated answer: 20000000
[T1] Calculated answer: 40000000
[T1] Calculated answer: 60000000
[T1] Calculated answer: 80000000
[T1] Calculated answer: 100000000
[T1] Aggregated of calculated answer: 20000000, 40000000, 60000000, 80000000, 100000000
[T7] File Content: Data from file..
[T7] Calculated answer in ReadFromFile: 20000000
[T7] Calculated answer in ReadFromFile: 40000000
[T7] Calculated answer in ReadFromFile: 60000000
[T7] Calculated answer in ReadFromFile: 80000000
[T7] Calculated answer in ReadFromFile: 100000000
[T7] File data returned.
[T7] FileData: File Content: Data from file.; Calcualted: 20000000, 40000000, 60000000, 80000000, 100000000
```

### 非同步開發的死鎖 (DeadLock) 如何產生與避免
#### 產生原因
在非同步開發中，死鎖通常源於同步等待非同步操作的結果。當使用 `await` 等待某個非同步呼叫時，會先將當下的執行緒上下文 (Context) 記起來，等之後非同步操作完成後再載入原先記起來的執行緒上下文繼續進行後續操作。而這個執行緒上下文指的是 `SynchronizationContext` 的物件 (Instance)，在傳統的 ASP.NET 環境中是 `AspNetSynchronizationContext` 這個子類別的物件。  
> 如果當下沒有 `SynchronizationContext` 可以拿，則會以 `TaskScheduler` 來決定後續的執行緒上下文。  
> 在 ASP.NET Core 之後，預設沒有 `SynchronizationContext`，因此不容易發生此類死鎖。

當呼叫端直接使用 `.Result` 或 `.Wait()` 來同步等待非同步操作時，這會阻塞目前的 `SynchronizationContext` 所關聯的執行緒，等待非同步操作完成。當非同步操作完成後，接著會想取得先前被記住的 `SynchronizationContext` 物件來繼續進行後續操作，但當時的這個物件已經被阻塞了。最終造成呼叫端阻塞的 `SynchronizationContext` 物件來等待非同步操作完成，而非同步操作完成後又需要取用已經被呼叫端阻塞的 `SynchronizationContext` 物件進而造成死鎖。

``` csharp
public class DeadlockExample
{
    public void MainMethod()
    {
        // 使用 .Result 進行同步等待，可能導致死鎖
        var result = AsyncMethod().Result;  
        Console.WriteLine(result);
    }

    public async Task<string> AsyncMethod()
    {
        await Task.Delay(1000);  // 模擬一個非同步操作
        return "Completed";
    }
}
```

以上範例在 ASP.NET 環境中運作時：
+ `MainMethod` 使用 `.Result` 同步等待 `AsyncMethod` 的結果，這會阻塞當下的`SynchronizationContext` 物件。
+ `AsyncMethod` 的 `await Task.Delay(1000)` 會暫停目前方法的執行將控制權交回呼叫端，此時的呼叫端因為是同步等待所以就這樣等著。
+ 當 1 秒結束後，`await` 試圖取用先前記住的 `SynchronizationContext` 物件，來讓呼叫端能繼續後續操作。
+ 但此時這個物件以被阻塞，導致雙方陷入相互等待的死鎖狀態。

#### 避免方式
##### 使用 `await` 避免同步 (`.Result` 和 `.Wait()`) 等待非同步操作
非同步操作應該使用 `await` 來進行等待，而不是使用 `.Result` 或 `.Wait()`。

##### 使用 `ConfigureAwait(false)`
在某些情況下，我們並不需要回到原本的執行緒上下文繼續執行後續操作。這時可以使用 `ConfigureAwait(false)` 來讓這個非同步操作不要取用先前記住的執行緒上下文，進而避免死鎖風險。
> 從官方文件可以看到，第一個參數名字是 `continueOnCapturedContext` ，加上文件說明也可以看出端倪。  

這個解法的缺點是，因為 `async` 的傳染性，所有的呼叫端都是非同步方法，他們都要加上 `ConfigureAwait(false)`，所以通常不是優先選項，但換個角度來看，當在開發套件時反而成為必要的機制，因為我們要假設使用者不一定會乖乖的使用 `await`，因此必須加上 `ConfigureAwait(false)` 來預防。

#### 小結
了解死鎖發生的原因主要是因為了解背後的主因能讓我們在寫非同步程式時更有自信，而避免方式分兩種情境：
1. 多數情境下，做為呼叫端永遠使用 `await`。  
2. 開發套件或給外部開發人員使用的程式時，使用 `ConfigureAwait(false)` 來預防呼叫端誤用。

### 結論
非同步程式是個很難完整表達的特性，只好借助 AI 來梳理並記錄，這樣之後需要說明的時候會更能有條理的表達。

### 參考
ChatGPT  
[.NET 本事：非同步程式設計](https://play.google.com/store/books/details/NET_%E6%9C%AC%E4%BA%8B_%E9%9D%9E%E5%90%8C%E6%AD%A5%E7%A8%8B%E5%BC%8F%E8%A8%AD%E8%A8%88_Alpha_%E7%89%88?id=AwS1DAAAQBAJ)