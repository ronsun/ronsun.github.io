---
title: 利用xs3p將xsd轉換成文件
date: 2017-09-02 15:40:30
categories:
- Tools

---

有時候專案中會需要使用xml做為設定文件去定義程式的行為與參數, 而config有可能會非常複雜, 所以為了開發上的方便會用xsd來定義, 並且需要產出一份文件來讓新手快速了解config應該如何編寫, 所以我們需要一些工具來幫來製作文件。

<!--more-->

目前這個專案, 需要文件化的xsd只有一個, 所以並沒有使用大砲級的 [Liquid Studio](https://www.liquid-technologies.com/xml-studio) 以及 [DocFlex](http://www.filigris.com/docflex/), 而是用了比較輕量簡單的 xs3p , 但是 [官方的xs3p](https://xml.fiforms.org/xs3p/) 排版太陽春, 所以這邊推薦使用 [Github上的美化版](https://github.com/bitfehler/xs3p), 或是我[後來 fork 出來改過的版本](https://github.com/ronsun/xs3p).  

接著會用 bitfehler 版本的 xs3p 介紹這個小工具的使用. 

## 下載工具與依賴元件
+ [xs3p美化版](https://github.com/bitfehler/xs3p)
+ [Command Line Transformation Utility (msxsl.exe)](https://www.microsoft.com/en-us/download/details.aspx?id=21714)
+ [MSXML 4.0 Service Pack 3](https://www.microsoft.com/en-us/download/details.aspx?id=15697): 可以不裝, 但轉換時Annotation中如果有CDATA區段會轉不出來。

## xs3p資料夾結構與使用

### 主要文件
+ `/examples`: 範例資料夾。
  - `/examples/test_*.bat` : 執行各種轉換工具的批次檔。
  - `/examples/*.xsd`: 範例XSD檔。
+ `/xs3p.xsl`: 轉換時需要使用的樣式定義。

### 使用(以msxml為例)
1. 將 `msxsl.exe` 放到 `/examples` 資料夾中。
2. 點擊 `/examples/test_msxsl.bat` 直接執行。
3. 會在 `/examples/msxsl-results` 中產出 `/examples` 目錄下所有XSD檔轉換後的html文件。
4. [範例XSD](sample.xsd) 與 [範例HTML](sample.html)。

## 各種問題

### 編碼問題  

轉出來的html檔所需要的css, js等資源都是連到cdn上拿, 如果在chrome上面樣式無法套用, 可能是編碼問題, 要在 xs3p.xsl 中的 `<link>` 元素中加上 `charset="UTF-8"`。  

```
<link href="{$bootstrapURL}/css/bootstrap.min.css" rel="stylesheet" charset="UTF-8"/>
```

### 命名空間問題  
用xs3p搭配msxml.exe去產生xsd文件時, 需要特別當一個命名空間設定多個前綴, 只認第一個。  

舉例, 下面的範例中, `xmlns:first`以及 `xmlns:me`是同樣的命名空間, 轉換時只認第一個, 所以如果xsd中以me為前綴的type轉換出來的超連結都會是失效的, 如附圖
``` xml
<?xml version="1.0" encoding="utf-8"?>
<xs:schema id="ComponentDefinition"
    targetNamespace="http://www.ronsun.com/ComponentDefinition/Guide.md"
    elementFormDefault="qualified"
    xmlns:first="http://www.ronsun.com/ComponentDefinition/Guide.md"
    xmlns:me="http://www.ronsun.com/ComponentDefinition/Guide.md"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
>

</xs:schema>
```
  
{% asset_img 001-duplicate-namespace.png %}  



### 列舉類型的Annotation無法轉換  
如下範例將註解加在enumeration中, 轉換後的文件會遺漏掉註解。
``` xml
  <xs:simpleType name="componentTypeEnum">
    <xs:restriction base="xs:string">
      <xs:enumeration value="Typ1">
        <xs:annotation>
          <xs:documentation>
            <![CDATA[This is type1]]>
          </xs:documentation>
        </xs:annotation>
      </xs:enumeration>
      <xs:enumeration value="Typ2">
        <xs:annotation>
          <xs:documentation>
            <![CDATA[This is type1]]>
          </xs:documentation>
        </xs:annotation>
      </xs:enumeration>
    </xs:restriction>
  </xs:simpleType>
```

因應方式是把註解加在simpleType下, 如:  
``` xml
  <xs:simpleType name="componentTypeEnum">
    <xs:annotation>
      <xs:documentation>
        <![CDATA[
        `Type1`: this is type1  
        `Type2`: this is type2
        ]]>
      </xs:documentation>
    </xs:annotation>
    <xs:restriction base="xs:string">
      <xs:enumeration value="Typ1" />
      <xs:enumeration value="Typ2" />
    </xs:restriction>
  </xs:simpleType>

```

## 後記
這個工具用起來很不錯, 不過如上面所說, 有一些細節不符合目前專案需求, 所以只好自己另外 fork 出來改成 [ronsun/xs3p](https://github.com/ronsun/xs3p).  
