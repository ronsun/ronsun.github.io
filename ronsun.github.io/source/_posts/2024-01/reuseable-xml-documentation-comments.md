---
title: 可重用的 XML 文件註解
date: 2024-01-22 23:52:49
categories:
- C#
- NET
tags:
---

在開發 C# 專案時，我們經常使用 [XML 文件註解](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/) 來為型別或成員寫說明，這類型的註解除了可以提高維護性外，也可以透過工具自動轉化成 API 文件。  

但是註解也是需要維護的，在多載方法的註解經常面臨一個問題就是一群多載方法通常有著相似的註解，維護時需要一個一個修改其實很容易使得最後不同多載間的註解有著許多細微的不一致。另一方面這樣也讓維護註解變得更麻煩瑣碎。

[XML 文件註解](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/) 官方文件很詳細，這篇文章不詳細說明這些內容，而會聚焦在重用註解的方式。

<!--more-->

### `<inheritdoc>`
首先是 `<inheritdoc>` 這個標記，這算是比較常用的部分，範例如下：
``` csharp
public class FluentUriBuilder
{
    /// <summary>
    /// Create <see cref="FluentUriBuilder"/>.
    /// </summary>
    /// <returns>Created <see cref="FluentUriBuilder"/>.</returns>
    public static FluentUriBuilder Create()
    {
        return new FluentUriBuilder();
    }

    /// <summary>
    /// <inheritdoc cref="Create()"/> With passed <paramref name="uri"/>.
    /// </summary>
    /// <param name="uri">Uri.</param>
    /// <returns>Created <see cref="FluentUriBuilder"/>.</returns>
    public static FluentUriBuilder Create(Uri uri)
    {
        return new FluentUriBuilder(uri);
    }
}
```

首先可以看到無參數方法 `Create()` 有一行註解 `Create <see cref="FluentUriBuilder"/>.`，對於另外一個多載來說應該也要套用相同的註解為基底，可能會需要加上針對該多載的特別描述，這時候就可以用 `<inheritdoc cref="Create()"/>` 的方式來 "繼承" 無參數方法的註解，並附加額外的說明。  

但這個方法有些缺點，首先，當多載數量一多的時候註解內容就會有很多繼承語句，可讀性不太好；其次，在決定要用哪個多載的註解當基底時也容易有標準不一致的情況。這兩個缺點都會讓註解的維護打了不少折扣。  

整體來說，堪用但可維護性偏差。

### `<include>`
同樣的程式碼片段，變成下面這樣：
``` csharp
public class FluentUriBuilder
{
    /// <summary>
    /// <include file='Properties/SharedComments.xml' path='SharedComments/Method[@name="FluentUriBuilder.Create"]'/>
    /// </summary>
    /// <returns>Created <see cref="FluentUriBuilder"/>.</returns>
    public static FluentUriBuilder Create()
    {
        return new FluentUriBuilder();
    }

    /// <summary>
    /// <include file='Properties/SharedComments.xml' path='SharedComments/Method[@name="FluentUriBuilder.Create"]'/>
    /// With passed <paramref name="uri"/>.
    /// </summary>
    /// <param name="uri">Uri.</param>
    /// <returns>Created <see cref="FluentUriBuilder"/>.</returns>
    public static FluentUriBuilder Create(Uri uri)
    {
        return new FluentUriBuilder(uri);
    }
}
```

這個做法直接把共用註解檔獨立出去成一個 XML 檔案存放在 `Properties` 中，如下：
``` xml
<SharedComments>
  <Method name="FluentUriBuilder.Create">
    Create <see cref="FluentUriBuilder"/>.
  </Method>
</SharedComments>
```

優點在於避免 `<inheritdoc>` 在方法間相互 "繼承" 所引發的混亂，但相對的缺點就是會需要額外的註解檔，且以可讀性差來說和 `<inheritdoc>` 不相上下。  

### 結論
雖然 XML 文件註解很簡單，但其實他有各式各樣的標記讓我們在維護程式過的過程就順便維護 API 註解，能為自動產生的 API 提供很大的幫助最大化降低額外維護文件的成本與風險。 而註解因為沒有編譯器的保護所以寫壞很難察覺，如果能活用 XML 文件註解的話也能更容易的維持高品質的註解。

### 參考
[XML documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/)