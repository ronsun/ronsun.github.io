---
title: 依字位 (grapheme) 處理字串
date: 2022-05-28 01:15:20
categories:
- C#
- .NET
tags:
---

以前在處理字串的時候, 不管是反轉或是取字元都沒考慮到有些語言的一個字母可能是由多個字組成的, 直到遇到問題.  

舉個越南文的例子, `ơ` 是由 `o` 和 `̛ ` (%cc%9b) 組成的, 所以處理的時候就容易有預料外的結果, 以字串反轉來說, `ơa` 反轉後變成 `a̛o`

<!--more-->

### 關於 Grapheme 與相關
這部分沒有仔細研究, 但是網路資源很多可以需要時知道細節時再查詢.  
+ [What's the difference between a character, a code point, a glyph and a grapheme?](https://stackoverflow.com/q/27331819/8223582)  
+ [Glossary of Unicode Terms](http://www.unicode.org/glossary/): 相關關鍵字 `Abstract Character` / `Character` / `Glyph` / `Grapheme`
  
### 使用 TextElementEnumerator 處理字串
使用 `TextElementEnumerator` 就能正確的將 `ơa` 反轉成 `ơa` 了.  

``` csharp
using System.Globalization;
public string ReverseGraphemeClusters(string s)
{
    TextElementEnumerator enumerator = StringInfo.GetTextElementEnumerator(s);
    StringBuilder sb = new StringBuilder();
    while (enumerator.MoveNext())
    {
        sb.Insert(0, enumerator.GetTextElement());
    }

    return sb.ToString();
}
```

### 結論
就是個沒遇到不會想到的問題, 處理方式也很簡單, 只是背後細節要看很多文件才能了解很麻煩所以乾脆紀錄一下備查.  

### 參考
[What's the difference between a character, a code point, a glyph and a grapheme?](https://stackoverflow.com/q/27331819/8223582)  
[Glossary of Unicode Terms](http://www.unicode.org/glossary/)  
[Correctly reversing a string](https://riptutorial.com/csharp/example/10627/correctly-reversing-a-string)  
[C# grapheme](https://zetcode.com/csharp/grapheme/)  