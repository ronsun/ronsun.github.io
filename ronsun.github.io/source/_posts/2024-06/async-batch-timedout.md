---
title: 在 C# 中控制非同步操作的總超時時間
date: 2024-06-23 23:53:35
categories:
- C#
tags:
---

在之前的專案中，我們遇到了一個情境，在需要處理多個非同步操作，例如多個網絡請求或文件處理時，需要設定一個總和的超時時間，當超過這個時間時，只回傳成功執行的非同步操作結果，並放棄還沒做完的部分。這篇文章會說明如何應對這種特殊需求。

<!--more-->

### 問題細節說明
對於批次非同步操作，一般有兩種策略：一是使用 `Task.WhenAll` 在所有操作都完成時繼續；二是使用 `Task.WhenAny` 在任一操作完成時繼續。但這兩種策略都不能解決這個問題：在特定時間範圍內只處理已完成的操作。

### 解決方案
使用 `Task.WhenAny` 搭配一個計時用的 Task 來達到目標。  

``` csharp
public class AsyncTimeoutExample
{
    public async Task<List<string>> ProcessTasksWithTimeout(IEnumerable<Task<string>> tasks, TimeSpan timeout)
    {
        // The key point is the timer task.
        var timeoutTask = Task.Delay(timeout);
        var taskList = new List<Task<string>>(tasks) { timeoutTask };
        var results = new List<string>();

        while (taskList.Count > 0)
        {
            Task completedTask = await Task.WhenAny(taskList);
            // Timed out, stop then return completed results.
            if (completedTask == timeoutTask)
            {
                break;
            }

            taskList.Remove(completedTask);
            if (completedTask is Task<string> resultTask)
            {
                // A task completed in time.
                // Keep the result and continue until timeout or every tasks are completed.
                results.Add(await resultTask);
            }
        }

        return results;
    }
}
```

### 結論
這個需求其實很特殊，一般情境下不會遇到，但使用一個計時用的 Task 搭配 `Task.WhenAny` 的策略很有趣，所以記錄一下，供未來參考。

### 參考
ChatGPT