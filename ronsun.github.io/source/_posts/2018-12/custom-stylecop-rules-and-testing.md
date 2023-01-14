---
title: 自訂 StyleCop 規則與測試方式
date: 2018-12-08 18:08:42
categories:
- Tools
tags:
---

很多 .NET 開發人員對於 StyleCop 應該是不陌生, 它能有效地協助我們讓程式碼的風格保持一致, 預設的規則其實已經足夠了, 但有些團隊可能有自已的特殊編碼規範, 這些編碼規範甚至是跟預設的規則是衝突的, 這時候就需要自己寫規則來套用.  

這篇主要是針對自訂規則可能會面臨的問題所記錄的, 尤其是要怎麼測試這些規則沒寫錯, 所以對於一些 google 就可以很容易找到的部分只會簡單帶過, 另外由於 StyleCop 也滿大的, 所以有些太細的設定或是內部運作這篇也不打算深入探索, 等之後有需要或是有興趣再根據各個主題或面向來專題探討.  

<!--more-->

### 安裝 StyleCop
安裝 StyleCop 的方式隨便問一下 google 大師就有很多教學了, 可以從 NuGet 找到 StyleCop.Analyzers 裝在各自專案上也可以從 Tools > Extensions and Updates 找到 StyleCop 裝到本機電腦上, 我會比較推薦從 Extensions and Updates 裡面裝, 因為只有需要裝一個 StyleCop 擴充元件就能夠套用到所有的專案上, 各專案只需要編輯各自的設定檔就可以了, 非常方便.  

安裝後的目錄在 `C:\Users\{username}\AppData\Local\Microsoft\VisualStudio\{version}\Extensions\{key}` 中, 其中 username 是本機使用者登入的名稱, version 是 visual studio 的版本號, key 是隨機產生的, 如果安裝很多擴充元件的話要一個個點進去找一下才知道哪個是 StyleCop 的.  

### 使用 StyleCop
都是 visual studio 上面點一點就好的, 教學資源也極多, 基本的使用就不多說了.  

#### 編輯 StyleCop 設定檔
編輯 StyleCop 設定檔的方式一般只要從 visual studio 上開啟圖形介面就可以編輯專案要用的設定了, 另外一種方式是使用 StyleCop 的安裝目錄 (如上一段所說的位置) 中的 StyleCop.SettingsEditor.exe 來編輯, 雙擊設定檔或是用命令提示字元下 `StyleCop.SettingsEditor.exe Settings.StyleCop` 指令就可以, 其中 Settings.StyleCop 是設定檔的路徑, 這個工具是用來編輯設定檔的, 所以不能建立一個新的設定檔, 要編輯的設定檔也必須是符合規定的格式, 不然讀取會錯誤.  

比起在 visual studio 中開啟圖形介面編輯設定檔, 用工具編輯雖然看起來不實用, 但是在需要製作很多份不同設定檔的時候卻很適合 (例如要測試自訂規則的驗證結果時), 這部分後面會說.  

> 如果覺得 StyleCop.SettingsEditor.exe 不能建立一個新的設定檔很不方便, 可以把原始碼 fork 出來, 自己改造成適用的小工具.  

### 自訂規則
自訂規則的部分步驟還不少, 而且很多情況下做錯也不會跳錯誤訊息, 所以花了不少時間在 try and error 上, [我在 GitHub 上面有放了一個簡單的自訂規則的專案 - ExtendedStyleCopRules](https://github.com/ronsun/ExtendedStyleCopRules), 這一段的說明會以這個專案的內容當作範例.  

#### 依賴組件
有幾個需要依賴的組件
+ StyleCop
+ StyleCop.CSharp

依賴的組件可以選擇從 NuGet 裝, 也可以直接依賴到前面從 Extensions and Updates 裝好的 StyleCop 的安裝目錄下, **這邊是建議直接從 NuGet 裝**, 因為 StyleCop 擴充元件的安裝資料夾名稱是隨機的, 換台電腦就找不到引用了, 很難管理.  

另外這邊要注意一下, 依賴的組件如果是從 NuGet 抓的, 必須跟從 Extensions and Updates 上安裝的 StyleCop 版號一致, 例如: 如果我從 Extensions 上安裝 StyleCop 6.0, 開發自訂規則時依賴的是從 NuGet 上面安裝的 StyleCop, 這樣編譯出來的組件無法讓 StyleCop 正確載入.  

#### 規則設定檔
要讓 StyleCop 能夠將自定規則的相關設定放在設定頁面, 需要一個 xml 格式的設定檔, 並將這個檔案屬性設定為編譯成 Embedded Resource, 這邊以 ExtendedNamingRules.xml 以及一張設定截圖為例.  

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

{% asset_img customer-rules-setting.png %} 

+ `xmlns="...."` : 非必要, 只是沒有自動提示 (IntelliSense) 要寫這些 XML 太難過了, 所以自己刻了一個 [SourceAnalyzer.xsd](https://github.com/ronsun/ExtendedStyleCopRules/blob/master/src/ExtendedStyleCopRules/SourceAnalyzer.xsd) 來引用.  
+ `Name="Extend Naming Rules"` : 這個 SourceAnalyzer 的名字, 會顯示在圖中 A 點.  
+ `<Description>Extended naming rules.</Description>` : 這個 SourceAnalyzer 的描述, 點擊圖中 A 點時會顯示在圖中 C 點的位置.  
+ `Name="PrivateFieldNamesMustStartWithUnderscore"` : 規則名字, 顯示在圖中 B 點, 必須是大寫字母開頭.  
+ `CheckId="EA1301"` : 規則代號, 圖中 B 點, 也是違反規則時顯示在結果視窗中的代號, 必須是兩個大寫字母加四個數字.  
+ `<Context>.....</Context>` : 提示訊息, 違反規則時顯示在結果視窗中的訊息之一.  
+ `<Description>Validates that names...</Description>` : 規則描述, 點擊圖中 B 點時會顯示在圖中 C 點的位置.  

> 其實這個 XML 可以很複雜, 這個範例只是最簡單的情境, StyleCop 的官方文件區也有一些簡易說明, 不過如果要完整的內容的話, 可能需要去整合 StyleCop 所有規則的設定檔, 或是直接爬程式碼分析 XML 的完整結構了.  

#### 程式碼
先上程式碼再來慢慢說明.  
``` csharp
[SourceAnalyzer(typeof(CsParser))]
public class ExtendedNamingRules : SourceAnalyzer
{
    public override void AnalyzeDocument(CodeDocument document)
    {
        CsDocument csdocument = (CsDocument)document;

        if (csdocument.RootElement != null && !csdocument.RootElement.Generated)
        {
            csdocument.WalkDocument(new CodeWalkerElementVisitor<object>(VisitElement));
        }
    }

    private bool VisitElement(CsElement element, CsElement parentElement, object context)
    {
        // not target element
        if (element.Generated ||
            element.ElementType != ElementType.Field ||
            element.AccessModifier != AccessModifierType.Private)
        {
            return true;
        }

        if (!element.Declaration.Name.StartsWith("_"))
        {
            AddViolation(element, Rules.PrivateFieldNamesMustStartWithUnderscore);
        }

        return true;
    }
}
```

StyleCop 是把原始碼拆解成 Element, Statement 和 Expression, 然後走訪所有節點並根據委派方法決定驗證邏輯與行為.  

這個類別需要使用 SourceAnalyzer 這個 Attribute 並繼承 SourceAnalyzer 後覆寫相關的分析方法 `void AnalyzeDocument(CodeDocument document)`, 接著 `WalkDocument(...)` 這邊會開始走訪所有程式碼, 然後根據委派方法 `VisitElement(...)` 來判定違規與否以及添加違規訊息等, 另外 WalkDocument 方法還有其他多載能另外多帶兩個參數, 是用來驗證 Statement 和 Expression 的委派方法.  

> 後來在 Element, Statement 和 Expression 外又多了一個 QueryClause, 看起來是針對 LINQ 語句做的, 例如 `var q = from prod in db.Prodcut where prod.Category == "shoes"`, WalkDocument 方法有一個多載提供委派方法 queryClauseCallback, 用途和其他委派應該差不多.  

在刻這些規則時發現, 對於複雜一點的規則很難下手, 其中一個原因是不知道 StyleCop 是把程式碼拆解成什麼樣子, 驗證時需要用的拆解後的程式碼片段也不知道從哪裡得到, 這時候一個方法是去看 StyleCop 的原始碼, 或是偷懶一點, 利用測試在 debug 模式下看 CsDocument 的完整結構與相對應的值, 測試的方式後面會提到.  

#### 測試自訂規則
這個測試的目的是測試自訂規則是不是能正確驗證程式碼, 比較像整合測試.  

這邊的範例需要依賴一些組件, 從 NuGet 安裝即可
+ StyleCop (必要)
+ NUnit (測試框架)
+ FluentAssertions (Assert 用的套件)

下面是主要的程式碼片段與說明  
``` csharp
[TestFixture()]
public class ExtendedStyleCopRulesTests
{
    [TestCaseSource(nameof(TestCase_RuleTest))]
    public void RuleTest(string checkId, StyleCopViolations expectedValidationResult)
    {
        // arrange
        Directory.CreateDirectory(Locations.ValidationResultDirectory);
        string validationResultPath = Path.Combine(Locations.ValidationResultDirectory, $"{checkId}.xml");

        string ruleSettingPath = Path.Combine(Locations.TestDataDirectory, checkId, $"{checkId}.StyleCop");
        string codePath = Path.Combine(Locations.TestDataDirectory, checkId, $"{checkId}.cs");

        CodeProject project = new CodeProject(
            checkId.GetHashCode(),
            Locations.BaseDirectory,
            new Configuration(null),
            0);

        StyleCopConsole console = new StyleCopConsole(
            ruleSettingPath,
            false,
            validationResultPath,
            Locations.AddInDirectory,
            false,
            Locations.BaseDirectory);

        console.Core.Environment.AddSourceCode(project, codePath, null);

        // act
        console.Start(new CodeProject[] { project }, true);

        StyleCopViolations actualValidationResult;
        using (Stream s = new FileStream(validationResultPath, FileMode.Open))
        {
            var serializer = new XmlSerializer(typeof(StyleCopViolations));
            actualValidationResult = (StyleCopViolations)serializer.Deserialize(s);
        }

        // assert
        bool testDataExist = File.Exists(ruleSettingPath) && File.Exists(codePath);
        testDataExist.Should().BeTrue();
        actualValidationResult.ViolationItems.Should().BeEquivalentTo(expectedValidationResult.ViolationItems);
    }

    private static List<object[]> TestCase_RuleTest()
    {
        var expectedValidationResult = new StyleCopViolations() { };
        return new List<object[]>()
        {
            EA1301TestResource()
        };
    }

    private static object[] EA1301TestResource()
    {
        string checkId = "EA1301";
        var expectedValidationResult = new StyleCopViolations()
        {
            ViolationItems = new List<StyleCopViolations.Violation>()
            {
                new StyleCopViolations.Violation()
                {
                    LineNumber = "7",
                    RuleNamespace = "ExtendedStyleCopRules.NamingRules.ExtendedNamingRules",
                    Rule = Rules.PrivateFieldNamesMustStartWithUnderscore.ToString(),
                    RuleId = checkId
                }
            }
        };
        return new object[] { checkId, expectedValidationResult };
    }
}
```

測試的部分是參考 StyleCope 原始碼中的測試專案, 其實主要功能 StyleCop 都有提供, 也就是上面程式碼中的 CodeProject 和 StyleCopConsole, 這一句 `console.Start(new CodeProject[] { project }, true)` 執行後會根據指定的程式碼檔案 (`codePath`) 以及套用的 StyleCop 規則 (`ruleSettingPath`) 健行驗證, 並將檢查結果輸出成 XML 存檔到指定位置 (`validationResultPath`), 接著我們再讀取檢查結果驗證是否符合預期.  

完整測試程式碼可以看 [GitHub](https://github.com/ronsun/ExtendedStyleCopRules/tree/master/src/ExtendedStyleCopRules.Tests)  

> 其實 StyleCop 的測試專案是連預期驗證結果都寫成 XML 了, 比這個複雜多了, 不過以自己的小專案來說, 還是先簡單就好, 等規則多到像 StyleCop 那樣再說 (應該不可能).  

#### 套用規則
編譯好, 把 dll 放到 StyleCop 擴充元件的安裝目錄下, 然後重開 visual studio 就可以了.  

### 結論
StyleCop 教學資源多集中在說明怎麼使用, 自訂規則的文章也不少, 但有提到測試的很少了, 所以乾脆直接跳進去看 StyleCop 內建的規則和測試是怎麼寫的, 順便整理了一下相關的 XML 規格刻個 XSD 來輔助使用, 而且測試的部分能協助我們在偵錯模式下去看被 StyleCop 拆解後的程式碼, 也更容易知道怎麼判斷程式碼是否違規.  

以往沒看套件原始碼的習慣, 雖然剛開始看覺得很有障礙, 不過當作觀摩練習也是不錯.  

### 參考
[Instant Stylecop Code Analysis How-to (書)](https://www.tenlong.com.tw/products/9781782169543?list_name=srh)  
[StyleCop](https://github.com/StyleCop/StyleCop)  
[How to Implement a Custom StyleCop Rule](https://sites.google.com/a/rees.biz/main/Home/customstylecoprules)  
[C# Code Reviews using StyleCop – Detailed Article](https://www.codeproject.com/Articles/30762/C-Code-Reviews-using-StyleCop-Detailed-Article?msg=2803242#StyleCopCodeparsinglogic)  
[Part I: Creating Custom Rules for Microsoft Source Analyzer](http://www.lovethedot.net/2008/05/creating-custom-rules-for-microsoft.html)  
[Part II: Creating Custom Rules for Microsoft Source Analyzer](http://www.lovethedot.net/2008/05/creating-custom-rules-for-microsoft_27.html)  
[Part III: Creating Custom Rules for Microsoft Source Analyzer](http://www.lovethedot.net/2008/05/creating-custom-rules-for-microsoft_6976.html)  

> 有些資料比較久了, 小細節和現在不一定相同, 不過說明很詳細, 對於了解 StyleCop 是很有幫助的

---
