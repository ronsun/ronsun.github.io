---
title: readonly 欄位與 struct 防禦性複製陷阱
date: 2026-04-24 18:14:38
categories:
- C#
- Language Spec
tags:
---

在看 Framework Design Guidelines 時，讀到官方把 `Nullable<T>` 改成 `readonly` 的過程中，曾意外引發一個不易察覺的問題。也因此我才注意到防禦性複製（defensive copy）這個議題，以及它帶來的陷阱。  

相關討論串在 [dotnet/corefx PR #24997](https://github.com/dotnet/corefx/pull/24997)

<!--more-->

### 防禦性複製
詳見外部參考 [C# struct 的防禦性複製（defensive copy）](https://www.huanlintalk.com/2026/03/csharp-defensive-copy.html)。  

### 問題展示

以 GitHub 的案例，完整程式碼 [`Nullable<T>`](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Nullable.cs) 裁剪後在 `value` 欄位加上 `readonly` 如下：

``` csharp
public struct FakeNullable<T> where T : struct
{
    // It can be mutated in ToString, etc.
    internal readonly T value;

    public FakeNullable(T value)
    {
        this.value = value;
    }

    public readonly T Value => value;

    public override string? ToString()
    {
        /* Defensive copy is what causes the issue here.
            IL_0000: ldarg.0
            IL_0001: ldfld !0 valuetype Program/FakeNullable`1<!T>::'value'
            IL_0006: stloc.0
            IL_0007: ldloca.s 0
            IL_0009: constrained. !T
            IL_000f: callvirt instance string [System.Runtime]System.Object::ToString()
            IL_0014: ret
        */
        return value.ToString();
    }
}

struct Counter
{
    public int Count;

    public override string ToString()
    {
        Count = 5;
        return "";
    }
}


FakeNullable<Counter> counter = new FakeNullable<Counter>(new Counter());
Console.WriteLine(counter.Value.Count); // 0
counter.ToString();
Console.WriteLine(counter.Value.Count); // Expect 5, but still get 0
```

原因在 `FakeNullable<T>.ToString()` 這行：

``` csharp
return value.ToString();
```

`value` 是 `readonly field`。對唯讀欄位呼叫 instance method 時，編譯器會先做一份防禦性複製，再對複本呼叫方法，概念上接近：

``` csharp
var temp = value;
return temp.ToString();
```

所以 `Counter.ToString()` 改到的是複本 `temp.Count`，不是原本的 `value.Count`。這就是為什麼 `counter.ToString()` 執行後，`counter.Value.Count` 仍然是 `0`。  

對照原本非唯讀版本產生的 IL Code，可以更清楚看出差異：  
``` csharp
public struct FakeNullable<T> where T : struct
{
    internal T value;

    public FakeNullable(T value)
    {
        this.value = value;
    }

    public readonly T Value => value;

    public override string? ToString()
    {
        /*
            IL_0000: ldarg.0
            IL_0001: ldflda !0 valuetype Program/FakeNullable`1<!T>::'value'
            IL_0006: constrained. !T
            IL_000c: callvirt instance string [System.Runtime]System.Object::ToString()
            IL_0011: ret
        */
        return value.ToString();
    }
}
```

### 結論

防禦性複製是編譯器時期額外加入的操作，不特別留意的話很容易忽視。

### 參考
AI Tools  

[Framework Design Guidelines: Conventions, Idioms, and Patterns for Reusable .Net Libraries, 3/e](https://www.tenlong.com.tw/products/9780135896464)  

[C# struct 的防禦性複製（defensive copy）](https://www.huanlintalk.com/2026/03/csharp-defensive-copy.html)  

[dotnet/corefx PR #24997](https://github.com/dotnet/corefx/pull/24997)  

[`Nullable<T>`](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Nullable.cs)