---
title: C# 語言特性更新 - C# 11
date: 2024-06-30 23:41:03
categories:
- C#
- Language Spec
tags:
---

C# 各版本新特性摘要，包含自己的想法與實務上的偏好。

<!--more-->

### 特性可支援泛型 (Generic attributes)
可在特性上使用泛型參數。
``` csharp
public class GenericAttribute<T> : Attribute { }

[GenericAttribute<string>()]
public string Method() => default;
```

但有一些限制：
1. 必須是已確定的型別，例如 `GenericAttribute<string>`；不能是不確定的泛型型別像是 `GenericAttribute<T>`。
2. 型別引數和 `typeof` 有相同的限制，不允許需要 metadata annotation 的型別，像是 `dynamic`、`string?` 等可為 null 的參考型別、`(int x, int y)` 等各種 Tuple。但上述情境分別可用 `object`、`string`、`ValueTuple<int, int>` 取代。

> 關於 metadata annotation 以及為什麼有這些限制應該找時間深入研究背後的機制。  

### Generic math support

### 結論

### 參考
