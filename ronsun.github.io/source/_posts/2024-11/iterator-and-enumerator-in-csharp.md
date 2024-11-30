---
title: C# 中迭代器的實現與列舉器
date: 2024-11-30 01:04:23
categories:
- C#
- Language Spec
tags:
---

迭代器模式（Iterator Pattern）是一種常見的設計模式，相關實作和說明在網路上隨處可見。在 C# 中，這個模式是藉由列舉器（Enumerator）來實現的。除了直接操作列舉器，C# 還提供了多種語法糖使開發更加便捷。本篇文章將不再贅述迭代器模式的理論，而是聚焦於 C# 提供的介面與語法糖，詳細說明列舉器在 C# 中的實際應用。

<!--more-->

### 從 `IEnumerable` 與 `IEnumerable<T>` 介面開始
在 C# 中，實現集合的遍歷功能主要透過實作 `IEnumerable` 或 `IEnumerable<T>` 介面來完成。`IEnumerable` 定義了一個 `GetEnumerator()` 方法，此方法會回傳一個列舉器，而列舉器負責實際的遍歷邏輯。從 [官方文件](https://learn.microsoft.com/en-us/dotnet/api/system.collections.ienumerable) 可以看到詳細內容。

直接操作這些物件對於呼叫端來說非常繁瑣。而 C# 提供了一些語法糖來大幅降低操作上的複雜度。

### 語法糖 `foreach`
#### 透過 `foreach` 簡化列舉器的操作
很多工程師都沒意識到 `foreach` 其實是語法糖，下面會透過一段很簡單的程式碼與反組譯後的結果來說明 `foreach` 的真實樣貌。

以下是原始程式碼：
``` csharp
var names = new List<string> { "Ron", "John" };
foreach (var name in names)
{
	Console.WriteLine(name);
}
```

這段程式碼看似簡單，但透過下面的反組譯結果顯示，實際執行的程式碼非常冗長。
``` csharp
List<string> list = new List<string>();
list.Add("Ron");
list.Add("John");
List<string> names = list;
using (List<string>.Enumerator enumerator = names.GetEnumerator())
{
	while (enumerator.MoveNext())
	{
		string name = enumerator.Current;
		Console.WriteLine(name);
	}
}
```

#### 可和 `foreach` 搭配使用的型別
現在我們知道 `foreach` 會簡化對列舉器的操作，但哪些型別的物件可以和 `foreach` 搭配使用呢？ 根據上一段的範例，我們知道只要實作了 `IEnumerable` 或 `IEnumerable<T>` 介面的類型，就能與 `foreach` 搭配使用。

但這個結論不完全涵蓋所有情境，事實上：  
> 只要類型中定義了 `GetEnumerator()` 方法，即便沒有實作上述介面，仍然可以與 `foreach` 搭配使用。

以下範例展示了這一情境：
``` csharp
public class CustomCollection
{
    private string[] items = { "Alice", "Bob", "Charlie" };

    public IEnumerator<string> GetEnumerator()
    {
        return new CustomEnumerator(items);
    }

    private class CustomEnumerator : IEnumerator<string>
    {
        private string[] _items;
        private int _position = -1;

        public CustomEnumerator(string[] items)
        {
            _items = items;
        }

        public string Current
        {
            get
            {
                if (_position < 0 || _position >= _items.Length)
                {
                    throw new InvalidOperationException();
                }
                return _items[_position];
            }
        }

        object IEnumerator.Current => Current;

        public bool MoveNext()
        {
            _position++;

            if (DateTime.Now.DayOfWeek == DayOfWeek.Sunday && _items[_position] == "Charlie")
            {
                return false; // 假設週日不回傳 "Charlie"
            }

            return (_position < _items.Length);
        }

        public void Reset()
        {
            _position = -1;
        }

        public void Dispose() { }
    }
}

public class Program
{
    public static void Main()
    {
        CustomCollection customCollection = new CustomCollection();
        foreach (var item in customCollection)
        {
            Console.WriteLine(item);
        }
    }
}
```

這種基於約定來確認類型是否定義特定方法 (`GetEnumerator()`) 方法的設計風格我個人是很不喜歡的，以應用程式開發來說設計風格需要依賴開發團隊內部約定，在長期維護時很難管理。  

到這邊我們已經了解如何利用 `foreach` 簡化列舉器的操作，但列舉器本身的實作仍然非常麻煩。下一節將介紹另一個語法糖來解決這個問題。

### 語法糖 `yield` 
#### 透過 `yield` 簡化列舉器的實作
手動實作 `IEnumerator` 的過程往往會使程式碼變得冗長且不直觀。為了解決這個問題，C# 提供了 `yield` 關鍵字，用來簡化列舉器本身的實作。

使用 `yield return` 可以讓我們一步步回傳集合中的元素，而 `yield break` 則可以用來終止迭代。編譯器會根據 `yield` 的使用自動產生像上一節介紹的 `CustomEnumerator` 那樣複雜的列舉器。

以下是一個使用 `yield` 的範例：
``` csharp
public class CustomCollection
{
    private string[] items = { "Alice", "Bob", "Charlie" };

    public IEnumerator<string> GetEnumerator()
    {
        foreach (var item in items)
        {
            yield return item;
        }
    }
}

public class Program
{
    public static void Main()
    {
        var customCollection = new CustomCollection();
        foreach (var item in customCollection)
        {
            Console.WriteLine(item);
        }
    }
}
```

如果我們不需要自定義集合型別，甚至可以簡化成下面程式碼，事實上這是多數情境的用法：
``` csharp
public class Program
{
    public static IEnumerable<string> GetNames()
    {
        yield return "Alice";
        yield return "Bob";

        if (DateTime.Now.DayOfWeek == DayOfWeek.Sunday)
        {
            yield break; // 假設週日不回傳 "Charlie"
        }

        yield return "Charlie";
    }

    public static void Main()
    {
        foreach (var name in GetNames())
        {
            Console.WriteLine(name);
        }
    }
}
```

### 結論
雖然 `foreach` 和 `yield` 等語法糖大幅簡化了我們對迭代器的操作與實作，但了解這些語法糖在編譯後所產生的實際程式碼，對深入掌握 C# 有很大的幫助。而這些深入的了解也能讓我們在除錯過程中避免許多誤解與疑惑。

### 參考
ChatGPT