---
title: 傳遞依賴 (transitive dependency)
date: 2021-05-23 02:20:58
categories:
- C#
- .NET Core
tags:
---

為了降低多個專案間的複雜度, 我們通常會謹慎的控制專案之間的依賴, 但在最近新開案的 .NET Core 一系列的新專案中發現傳遞依賴會讓專案之間產生預期外的依賴.  

<!--more-->

傳遞依賴 (transitive dependency) 指的是, 當 A 專案依賴 B 專案, 而 B 專案依賴 C 專案時, A 專案會傳遞依賴 C 專案 (即在 A 專案中能存取 C 專案的公開成員), 用下面的圖當範例說明 

 {% asset_img dependency.jpg %}   

### 專案間的傳遞依賴
就上圖的例子, 現在三層都是我們自己建立的專案, WebAPI 依賴 Service, 而 Service 依賴 Data Access, 本來不希望能從 WebAPI 中直接存取到 Data Access, 但卻因為傳遞依賴而失算.  

現在直接依賴的關係是這樣的:  
> WebAPI (project) --> Service (project) --> Data Access (project)

解決方式是要在 **WebAPI 專案的 csproj 檔案中**將傳遞依賴關閉, 如下: 
``` xml
<Project>
  <PropertyGroup>    
    <DisableTransitiveProjectReferences>true</DisableTransitiveProjectReferences>
  </PropertyGroup>
</Project>
```

### 專案間和套件的傳遞依賴

這個情境的直接依賴的關係是這樣的, 假設 Data Access 是一個套件 (就叫 DataAccess 好了), Service 透過 NuGet 安裝並使用他  
> WebAPI (project) --> Service (project) --> Data Access (Package)

這時候 `<DisableTransitiveProjectReferences>` 是沒有用的, 解決方式就變成要在 **Service 專案的 csproj 檔案中** 用 `<PrivateAssets>` 排除, 如下
``` xml
<PackageReference Include="DataAccess" Version="1.1.0">
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

關於 `<PackageReference>` 更細緻的設定可以看[這裡官方相關的介紹](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#controlling-dependency-assets).  

### 套件間的傳遞依賴
最後一種情況, 套件之間的傳遞依賴, 現在 Service 也是個套件, 但他依賴 Data Access  
> WebAPI (project) --> Service (Package) --> Data Access (Package)

其實這個跟上個情境是一樣的, 就是在 Service 中加上 `PrivateAssets`, 但如果們是站在 WebAPI 專案開發者而不是套件 (本例中的 Service) 開發者的角度, 就目前所知是無能為力, 只能期待後面維護的人不要誤用.  

### 真實情境
現實中遇到的情境是這樣的, 某項任務中要用到一個公司內部的套件, 而這個套件依賴了一個 JSON 的序列化套件 `Jil`, 那問題在於 Visual Studio 的提示功能可能會引導開發者使用 `Jil` 提供的類別, 但我們專案內部其實是用 `Json.NET`, 如果維護的人不小心就跟著 Visual Studio 的提示用了, 就變成一個功能用兩種套件處理而容易亂.  

另外一個疑慮是, 就算我們真的也是用 `Jil`, 如果有天我們移除對這個內部套件的依賴, 或是更版後內部套件不使用 `Jil` 了, 那就會讓我們因為升版一個套件而造成另一個套件的使用問題, 雖然不難解決, 但多個疑慮總不是好事.  

還有第三個疑慮最麻煩, 今天如果使用 A 和 B 兩個套件, 而他們分別依賴不同版本的 `Jil` 的時候就很糟了, 稍微用簡單情境實測一下是**會使用高版的 `Jil`**, 這意味著, 當你依賴 A 套件並透過傳遞依賴使用 `Jil` 時, 可能會因為後來依賴 B 套件而讓 `Jil` 被升版.  
**而 A 套件在執行階段其實是呼叫到被升版的 `Jil`, 這會使得 A 套件的執行結果可能被改變或因為新版 `Jil` 的 breaking changes 而在執行階段拋出例外**. 

綜合以上疑慮, 可以考慮不要透過傳遞依賴而是直接依賴該套件, 這樣版本衝突時在安裝套件過程 **有 機 會** 能發現.  
> 是的, 只是有機會能發現, 如果專案直接依賴高版本, 傳遞依賴低版本, 是不會警示的, 這種情境下最後還是會使用高版本的套件, 上面的疑慮還是無法獲得解決.  

套件版本衝突的問題在 {% post_link dependent-assembly-issue %} 這邊已經遇過, 而且這還是直接依賴的還算好掌握, 如果是透過傳遞依賴使用的掌握上就困難了.  

[我在 GitHub 上有放一個專案來展示這些疑慮](https://github.com/ronsun/BlogDemo.TransitiveDependency).  

### 其他可能的做法
主要是參考這篇 [NuGet > Dependencies](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/dependencies).  

####  限定套件版本範圍
如範例: 
```
<!-- Accepts 1.0 up to 1.x, but not 2.0 and higher. -->
<PackageReference Include="ExamplePackage" Version="[1.0,2.0)" />

<!-- Accepts exactly 1.0. -->
<PackageReference Include="ExamplePackage" Version="[1.0]" />
```

這個想法是限定套件版本讓版號衝突時直接失敗, 但是這樣可能會導致多個套件很難一起運作,  **而且官方不建議**.  

#### 用一個獨立專案管理套件依賴
這個之前有實驗過, 就是開一個 Shared 專案, 他會被很多專案依賴, 而套件依賴都在 Shared 專案內管理再讓依賴他的其他專案們都能使用一致的套件組, 但是副作用是其他專案可能會存取到設計上不允許存取的套件, 以下面為例: 
 
 > WebAPI (project) --> Service (project) --> Data Access (project) --> Shared (Project)  
 > Service (project) --> Shared (Project)  
 > WebAPI (project) --> Shared (Project)  

 如上的多層式設計, 且 Shared (Project)  --> Entity Framework Core (Package), 這時候很容易有人跳過中間幾層直接從 WebAPI 層用 Entity Framwork Core 存取資料庫, 所以需要依照真實情境去部分限縮傳遞依賴, 比較麻煩.  


### 結論
接下來可能會做一些內部套件類的專案, 還是要稍微注意一下傳遞依賴的問題, 避免讓使用者依賴到沒必要依賴的套件引發維護上的困擾.   

而在使用套件時也應該要更謹慎, 以用最少量的多功能套件解決最多的問題, 不然引發的問題都是很難發現的, breaking changes 造成執行階段才拋例外已經不是好現象了, 萬一問題是得到錯誤的結果而非拋錯那真的會哭哭.  

### 參考
[Transitive project references (`ProjectReference`)](https://stackoverflow.com/a/60852224/8223582): 這篇很完整

[Controlling dependency assets]((https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#controlling-dependency-assets))  

[NuGet > Dependencies](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/dependencies)