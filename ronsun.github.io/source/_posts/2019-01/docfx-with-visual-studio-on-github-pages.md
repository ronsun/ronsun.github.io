---
title: 整合 DocFx 和 Visual Studio 並佈署上 GitHub Pages
date: 2019-01-24 22:29:25
categories:
- Tools
tags:
---

DocFx 是一個很方便的專案文件製作工具, 能將 [XML 文件註解](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/xmldoc/xml-documentation-comments)轉換成靜態網頁, 也可以轉化其他另外寫的 Markdown 格式的文件, 這篇主要的目的是紀錄如何將 DocFx 與 Visual Studio 整合, 在編譯後產出專案文件並佈署到 GitHub Pages.  

順便提一下, 適用情境是在 visual stuiod 上開發 .NET 專案, 並且希望能統一管理, 其他情境不一定適合這樣整合.  

<!--more-->

### 如何建立
**建立工具專案**  
建一個獨立的工具專案用來放 DocFx 需要的套件, 刻意跟產品專案分開的目的是希望文件建立工具與產品程式碼不要混在同一個專案, 另外這個工具專案不含程式碼但要是可以在 visual studio 中編譯的, 所以選用 Class Library.  

專案結構與 Demo 程式碼如下圖 (TryDocFx.DocFx 是工具專案, 而 TryDocFx.ClassLib 是產品專案):  
 {% asset_img docfx-projects.png %}  

由於目標是能簡單佈署文件到 GitHub Pages 上, 所以這邊是[用 docs 資料夾來做為 GitHub Pages 的根目錄](https://help.github.com/articles/configuring-a-publishing-source-for-github-pages/), 而不是 gh-pages 分支, 也因此資料夾結構是如下面所示: 

 ```
TryDocFx
    |---- docs
    |---- src
           |---- TryDocFx.ClassLib
           |---- TryDocFx.DocFx
```

**安裝 docfx.console**  
從 Nuget 中找到 docfx.console 並安裝在上一步建立好的工具專案 (TryDocFx.DocFx) 中, 安裝完後, 會自動在工具專案中建立必要的檔案, 如下圖:  
 {% asset_img docfx-installed-content.png %}  

**修改設定**  
設定檔是 docfx.json, [官方文件](https://dotnet.github.io/docfx/tutorial/docfx.exe_user_manual.html#3-docfxjson-format)有提供這個設定檔的說明, 以本例來說需要修改幾個地方.  

1. 因為產品程式碼不在目前專案中, 所以需要需要指定產品程式碼來源: `"src": "../TryDocFx.ClassLib"`.  
2. 套件預設會把靜態網頁放在目前專案的 _site 資料夾下, 因為我們要佈署到 GitHub Pages 上, 所以需要修改輸出路徑: `"dest": "../../docs"`.  
3. exclude 的部分, 由於本例輸出路徑和工具專案是分開的, 所以可以從 exclude 設定中將 `"_site/**"` 拿掉. 

> 其實整個設定檔應該可以更簡潔, 例如說在本文的情境中 exclude 節點可能是可以移除的, 不過細節要一一研究太花時間了, 如果之後工作上有用到再來詳細了解設定檔的完整內容吧.  

修改後範例如下:
``` json
{
  "metadata": [
    {
      "src": [
        {
          "files": [
            "*.csproj"
          ],
          "cwd": ".",
          "exclude": [
            "**/obj/**",
            "**/bin/**"
          ],
          "src": "../TryDocFx.ClassLib"
        }
      ],
      "dest": "obj/api"
    }
  ],
  "build": {
    "content": [
      {
        "files": [
          "api/**.yml"
        ],
        "cwd": "obj"
      },
      {
        "files": [
          "api/*.md",
          "articles/**.md",
          "toc.yml",
          "*.md"
        ],
        "exclude": [
          "obj/**" 
        ]
      }
    ],
    "resource": [
      {
        "files": [
          "images/**"
        ],
        "exclude": [
          "obj/**"
        ]
      }
    ],
    "overwrite": [
      {
        "files": [
          "apidoc/**.md"
        ],
        "exclude": [
          "obj/**"
        ]
      }
    ],
    "dest": "../../docs",
    "template": [
      "default"
    ]
  }
}
```

**編譯專案**
改好設定後, 只要在 visual studio 中編譯 TryDocFx.DocFx 這個工具專案, 他就會根據設定將產品專案的 XML 文件註解轉換成靜態網頁並輸出到設定的輸出路徑中 (../../docs).  
最後只要設定好 GitHub Pages 並將專案全部推上 GitHub 就完成了.  

### 結論
可以將 C# 中的 XML 文件註解自動轉換成文件的方式很多, 不過如果要同時能整合 visual studio 又能包含自己寫的 Markdown 文件, 還得方便佈署的話, DocFx 是不錯的選擇.  

而從設定檔的結構與內容不難看出, 他的功能應該更強大, 不過這就等之後遇到複雜的情境時在來仔細研究了.  

### 參考
[DocFx](https://dotnet.github.io/docfx/index.html)

---