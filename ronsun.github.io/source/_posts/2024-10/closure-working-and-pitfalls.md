---
title: C# 中的閉包與陷阱
date: 2024-10-06 02:06:41
categories:
- C#
- Language Spec
tags:
---

閉包（Closure）是 C# 中的一種常用特性，允許方法內部引用其外部作用域的變數，進而提高程式的靈活性。然而，如果對閉包及變數捕捉（Capture）機制的理解不夠深入，可能會導致難以預期的錯誤。本文將介紹閉包與變數捕捉機制，並展示常見的陷阱及解決方案。

<!--more-->

### 閉包與捕捉
#### 以實值型別的變數捕捉介紹
閉包允許匿名方法、lambda 表達式或區域方法捕捉外部作用域的變數，即使在變數的作用域外，仍然可以引用並使用這些變數。

``` csharp
public class Demo
{
    public void Do()
    {
        int i = 0;

        Action<string> echo = input =>
        {
            i++;
            $"{input} - {i}".Dump();
        };
        
        // output "a - 1"
        echo.Invoke("a");
        
        i++;

        // output "2"
        i.Dump();
        
        // output "b - 3"
        echo.Invoke("b");
    }
}
```

在這個例子中，變數 `i` 是在 `Do()` 方法的作用域中宣告的，但透過閉包的捕捉機制，該變數可以在 `echo` 這個委派方法中被使用，而且當 `Do()` 方法中對 `i` 進行修改時，這個改變也會反映到 `echo` 中。

這背後的機制是編譯器的「顯示類別」（display class）。在編譯過程中，編譯器會自動生成一個輔助類別，將捕捉的變數作為該類別的欄位，且委派方法的內容也可能被捕捉進這個輔助類別中，即便這些變數是實值型別，它們也能跨越作用域進行共享。

以下是使用反組譯工具觀察編譯後的程式碼，能幫助我們更好地理解閉包的工作方式：
``` csharp
using System;
using System.Runtime.CompilerServices;
using RoslynPad.Runtime;

public class Demo
{
    [CompilerGenerated]
    private sealed class <>c__DisplayClass0_0
    {
        public int i;

        [System.Runtime.CompilerServices.NullableContext(1)]
        internal void <Do>b__0(string input)
        {
            i++;
            DefaultInterpolatedStringHandler val = default(DefaultInterpolatedStringHandler);
            ((DefaultInterpolatedStringHandler)(ref val))..ctor(3, 2);
            ((DefaultInterpolatedStringHandler)(ref val)).AppendFormatted(input);
            ((DefaultInterpolatedStringHandler)(ref val)).AppendLiteral(" - ");
            ((DefaultInterpolatedStringHandler)(ref val)).AppendFormatted<int>(i);
            ObjectExtensions.Dump(((DefaultInterpolatedStringHandler)(ref val)).ToStringAndClear());
        }
    }

    public void Do()
    {
        <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
        <>c__DisplayClass0_.i = 0;
        Action<string> echo = new Action<string>(<>c__DisplayClass0_.<Do>b__0);
        echo("a");
        <>c__DisplayClass0_.i++;
        ObjectExtensions.Dump(<>c__DisplayClass0_.i);
        echo("b");
    }
}
```

可以看到，變數 `i` 和委派方法 `echo` 都被捕捉進去了一個編譯器生成的類別（參考型別） `<>c__DisplayClass0_0` 中，這使得變數 `i` 可以在不同作用域之間共享。

> 為什麼了解這個機制很重要？  
> 對於初階工程師來說，對實值型別與參考型別特性的理解還不夠深入，可能會因為閉包捕捉變數的方式而產生誤解。例如，將實值型別與參考型別混淆，誤以為 `int` 是參考型別。  
> 這也是為什麼我常在工作中強調，Debug 時最好先排除沒問題的部分並簡化程式來減少干擾因素，這樣才能更準確地找出問題的根本原因，且要試著證明找到的根本原因是真正的問題，否則可能會導致看似修好 Bug，卻得到錯誤的結論，甚至對基本概念產生錯誤認知。  

#### 參考型別變數的捕捉

當捕捉的變數是參考型別時，閉包的行為是如何呢？讓我們看一個範例：
``` csharp
public class Model
{
    public int Number { get; set; }
}

public class Demo
{
    public void Do()
    {
        Model model = new Model();

        Action<string> echo = input =>
        {
            model.Number++;
            $"{input} - {model.Number}".Dump();;
        };
        
        // output "a - 1"
        echo.Invoke("a");
        
        model.Number++;

        // output "2"
        model.Number.Dump();
        
        // output "b - 3"
        echo.Invoke("b");
    }
}
```

在這裡，`model` 是一個參考型別的變數。透過反組譯工具觀察，編譯器同樣會產生一個類別來捕捉這個參考型別變數：
``` csharp
public class Demo
{
	[CompilerGenerated]
	private sealed class <>c__DisplayClass0_0
	{
		public Model model;

		[System.Runtime.CompilerServices.NullableContext(1)]
		internal void <Do>b__0(string input)
		{
			model.Number++;
			DefaultInterpolatedStringHandler val = default(DefaultInterpolatedStringHandler);
			((DefaultInterpolatedStringHandler)(ref val))..ctor(3, 2);
			((DefaultInterpolatedStringHandler)(ref val)).AppendFormatted(input);
			((DefaultInterpolatedStringHandler)(ref val)).AppendLiteral(" - ");
			((DefaultInterpolatedStringHandler)(ref val)).AppendFormatted<int>(model.Number);
			ObjectExtensions.Dump(((DefaultInterpolatedStringHandler)(ref val)).ToStringAndClear());
		}
	}

	public void Do()
	{
		<>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
		<>c__DisplayClass0_.model = new Model();
		Action<string> echo = new Action<string>(<>c__DisplayClass0_.<Do>b__0);
		echo("a");
		<>c__DisplayClass0_.model.Number++;
		ObjectExtensions.Dump(<>c__DisplayClass0_.model.Number);
		echo("b");
	}
}
```

### 常見的閉包陷阱
#### 迴圈中的變數捕捉
閉包常見的陷阱之一是迴圈中的變數捕捉，這可能會導致意外的行為：
``` csharp
List<Action> actions = new List<Action>();

for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}

foreach (var action in actions)
{
    action.Invoke();
}
```

在這段程式碼中，所有輸出的結果都是 5，因為閉包捕捉的是變數 `i` 的引用，而非當下迴圈中的值。當迴圈結束時，`i` 的值已經變成了 5，因此每個閉包在執行時，都會使用同一個 `i` 的引用。

解決方案是將迴圈變數的當前值複製到一個區域變數中：
``` csharp
List<Action> actions = new List<Action>();

for (int i = 0; i < 5; i++)
{
	int temp = i;
    actions.Add(() => Console.WriteLine(temp));
}

foreach (var action in actions)
{
    action.Invoke();
}
```

這樣，閉包捕捉的是區域變數 `temp` 的值，而不是變數 `i` 的引用。編譯後的程式碼顯示，這個區別導致了不同的行為。

錯誤範例編譯後的程式碼：
``` csharp
List<Action> actions = new List<Action>();
<>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
<>c__DisplayClass0_.i = 0;
while (<>c__DisplayClass0_.i < 5)
{
	actions.Add(new Action(<>c__DisplayClass0_.<Do>b__0));
	<>c__DisplayClass0_.i++;
}
foreach (Action action in actions)
{
	action();
}
```

正確範例編譯後的程式碼：
``` csharp
List<Action> actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
	<>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
	<>c__DisplayClass0_.temp = i;
	actions.Add(new Action(<>c__DisplayClass0_.<Do>b__0));
}
foreach (Action action in actions)
{
	action();
}
```

可以看到相較於原本錯誤範例中只建立一個物件實體，因為有 `temp` 的加入使得編譯後的程式碼會建立五個不同的物件，所以五個物件的 `temp` 值就是正確的 0 ~ 4。

### 結論
由於變數捕捉是編譯時期的語法糖，對其工作原理的誤解可能導致難以預期的錯誤。除了委派的參數捕捉外，所有能捕捉外部作用域變數的機制，都必須注意變數在不同作用域間的修改問題。雖然閉包與變數捕捉的錯誤表現形式有很多，但核心問題往往在於跨作用域對變數的不當修改。只要能避免這類操作，閉包通常不會造成問題。

### 參考
ChatGPT