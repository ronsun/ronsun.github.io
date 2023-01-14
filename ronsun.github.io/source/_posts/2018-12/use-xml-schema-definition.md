---
title: XSD 的使用與應用
date: 2018-12-25 01:30:30
categories:
- XML
- XSD
tags:
---

XSD 的文章很早之前就寫過, 但是當時剛開始寫部落格, 所以雖然寫了三篇但內容沒有編排又很亂, 因此最近想說稍微重寫一下, 留下重點就好.  

<!--more-->

### 前言
XSD 全名為 XML Schema Definition, 用來定義並驗證一個 XML 文件的格式, 除此之外他對我們在 Visual Studio 中編寫 XML 的時候能提供很大的幫助如下: 
+ 在 Visual Studio 中檢查 XML 的格式與內容是否符合定義. 
+ 在 Visual Studio 中使用自動提示 (Intellisence) 的功能來提示我們該怎麼寫. 
+ 移到自動提示的選項上還會顯示這個選項的註解(當然要先定義在 XSD 中), 滑鼠移過到上面也會跳出註解, 就像平常開發在呼叫 API 會有的自動提示和註解提示一樣.

因此當一個專案需要依賴大量的 XML 設定檔去設定或是設定檔本身比較複雜的時候, 定義一份完整的 XSD 能讓設定檔在編寫時能有更充足的提示, 不用一直花時間去翻設定說明, 也可以避免花時間去抓因手誤造成的bug.  

本文會用之前為自訂 StyleCop 的規則需寫的 XML 所做的 XSD - [SourceAnalyzer.xsd](https://github.com/ronsun/ExtendedStyleCopRules/blob/master/src/ExtendedStyleCopRules/NamingRules/ExtendedNamingRules.xml) 來做
範例. 

### 建立一個新的 XSD 檔
在 Visual Studio 中開啟一個專案, 並加入一個 XML Schema 檔案, 預設內容如下

``` xml
<?xml version="1.0" encoding="utf-8"?>
<xs:schema id="XMLSchema1"
    targetNamespace="http://tempuri.org/XMLSchema1.xsd"
    elementFormDefault="qualified"
    xmlns="http://tempuri.org/XMLSchema1.xsd"
    xmlns:mstns="http://tempuri.org/XMLSchema1.xsd"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
>
</xs:schema>
```

其中的幾個屬性說明如下: 
+ targetNamespace: 目前文件命名空間. 
+ elementFormDefault: 選填, qualified 或 unqualified, 預設是unqualified, 建議用 qualified. 
+ xmlns / xmlns:mstns / xmlns:xs: 各種引用的 XSD 與命名空間, 其中 http://www.w3.org/2001/XMLSchema 就是用來定義xsd的xsd schema, 這也是為什麼在 visual studio 中編寫 xsd 時能有自動提示與驗證. 

雖然預設的命名空間看起來是一個 URL, 但實際上命名空間只是用來做識別用的, 並非必要是 URL, 用 URL 的好處是可以提示使用者連上該網站得到更詳細的說明, 如下面範例.  

> URL 應該是放專案的 GitHub Pages 連結比較好, 不過這個專案我沒有建 GitHub Pages 所以就直接放 GitHub 專案的連結了.  

```
<xs:schema id="SourceAnalyzer"
    targetNamespace="https://github.com/ronsun/ExtendedStyleCopRules"
    elementFormDefault="qualified"
    xmlns:this="https://github.com/ronsun/ExtendedStyleCopRules"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
>
</xs:schema>
```

### 套用到 XML 上
#### 如何套用
XSD 寫好後, 在 Visual Studio 中點選 **XML > Schemas** 可以看到這個 XSD 已經被自動套用了, 接著建立一個新的 XML 檔並指定 xmlns 為剛剛建立的 XSD 的 targetNamespace 就可以套用到該 XML 檔上了.  

``` xml
<?xml version="1.0" encoding="utf-8" ?>
<SourceAnalyzer Name="Extend Naming Rules" xmlns="https://github.com/ronsun/ExtendedStyleCopRules">
  <Description>
    Extended naming rules.
  </Description>
  <Rules>
    <Rule Name="PrivateFieldNamesMustStartWithUnderscore" CheckId="EA1301">
      <Context>Private filed name must start with Underscore</Context>
      <Description>Validates that names of private field must start with an underscore.</Description>
    </Rule>
  </Rules>
</SourceAnalyzer>
```

#### 自動提示 (Intellisence) 與檢查
套用上之後就可以有自動提示與檢查功能.  

**不合規則時會有警示**
不合規則時, 右側卷軸會有一個淡藍色小點, 輸入的文字本身下面也會有一條淡藍色毛毛蟲, 滑鼠移過去可以看到詳細的警告描述, 且在 Error List 地方也會有警告, 非常方便.  
{% asset_img xsd-warning.png %}  

**自動提示功能**
選到提示的選項時, 會彈出預先在 XSD 中寫好的註解.  
{% asset_img xsd-intellisence.png %}  

滑鼠移過文字時, 會彈出預先在 XSD 中寫好的註解.  
{% asset_img xsd-annotation.png %}  

#### 在程式碼中套用檢查規則
雖然 Visual Studio 會顯示警告, 但總有些時候有人會自動忽略警告, 無法提早發現錯誤, 這時候還可以在程式中檢查, 在應用程式啟動時, 直接呼叫驗證方法, 如果有違反就直接拋例外, 做為最後一道防線.  

**但其實我是覺得這一段程式碼沒必要, 應該是要參考 Warning 的資訊, 並盡可能不要讓 Warning 存在而不是忽略它才是比較嚴謹的做法.**  

大致的程式碼如下, 邏輯上就是載入 XSD 與相關的 XML, 接著透過 `XDocument.Validate(...)` 方法來驗證.  

``` csharp
public static void CheckXML()
{
    var currentAssembly = Assembly.GetExecutingAssembly();

    XmlSchemaSet schemas = new XmlSchemaSet();
    Stream xsdStream = currentAssembly.GetManifestResourceStream("ronsun.github.io.lab.SourceAnalyzer.xsd");
    using (var reader = XmlReader.Create(xsdStream))
    {
        schemas.Add("https://github.com/ronsun/ExtendedStyleCopRules", reader);
    }

    using (Stream xmlStream = currentAssembly.GetManifestResourceStream("ronsun.github.io.lab.ExtendedNamingRules.xml"))
    {
        XDocument.Load(xmlStream).Validate(schemas, (sender, e) =>
        {
            // Actions for validate results
            if (e.Severity == XmlSeverityType.Error)
            {
                throw new Exception(e.Message);
            }
        });
    }
}
```

### 學習資源
XSD 相關的學習資源如下:  
[tutorials point](https://www.tutorialspoint.com/xsd/index.htm)  
[w3schools](https://www.w3schools.com/xml/schema_intro.asp) (en)  
[w3schools](http://www.w3school.com.cn/schema/index.asp) (zh-cn)  

### 結論
其實 XSD 不算很常用的功能, 有些大型專案需要的 XML 也不一定有 XSD, 在寫設定檔的時候就只能依賴文件跟記憶力, 但如果時間允許, 我還是贊成補上 XSD 文件的, 一方面可以讓後面的人上手比較容易, 另一方面 XSD 本身也扮演著文件的角色了, 就比較不需要另外去維護文件來說明 XML 應該怎麼寫.  

不過, XSD 本身做為文件可讀性不好, 之前我也 [fork 出一個開源專案 xs3p 做了一些修改](https://github.com/ronsun/xs3p), 可以將 XSD 傳換成 html 文件, 細節也曾經寫過一篇文章 {% post_link xsd-to-document-by-xs3p %} 記錄. 

### 參考
[How to: Validate Using XSD (LINQ to XML) (C#)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/how-to-validate-using-xsd-linq-to-xml)  

---
