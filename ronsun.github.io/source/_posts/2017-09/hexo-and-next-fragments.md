---
title: Hexo 與 hexo-theme-next 的零碎片段
date: 2017-09-02 19:44:42
categories:
- Hexo
tags:
---

Hexo以及hexo-theme-next樣板在使用上有滿多細節跟小地方是需要一直找資料解決的, 但是每個小東西都寫一篇覺得太零散了, 所以把相關的片段都放在這裡。

<!--more-->

## Hexo

### skip_render
hexo可以透過skip_render參數讓特定檔案不被渲染, 就以資產資料夾為例吧。  

首先直接建立一個新的Post, 然後將sample.html檔案放到render_sample資料夾下, 這時候資料夾結構看起來是這樣的
```
|-- _posts/2017-09/
|   |-- render_sample.md
|   |-- render-sample
|   |   |-- sample.html
```

接著`hexo g` 一下, 這時候靜態文章的目錄會變成這樣
```
|-- render-sample
|   |-- sample.html
|   |-- index.html
|   |-- sample
|       |-- index.html
```

**這就是問題點, sample.html應該純粹作為資產使用才對, sample資料夾不應該存在的。**  

所以我們必須找出hexo的_config.yml檔, 並且設定skip_render參數, 指定資產資料夾不被渲染
>`skip_render: _posts/*/*/**`

接著再重新`hexo g`一下, 目錄就可以正確的產出了
```
|-- render-sample
|   |-- sample.html
|   |-- index.html
```

這邊引述一下hexo官方文件的說明:
> skip_render: 跳過指定檔案的渲染，您可使用 glob 表達式 來配對路徑。

## hexo-theme-next