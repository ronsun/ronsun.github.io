---
title: NUnit 中一次性初始化的涵蓋範圍
date: 2020-06-06 23:23:31
categories:
- Tools
tags:
---

一般來說套件的使用方式沒什麼好寫的, 但是使用 NUnit 的 OneTimeSetUp 時, 究竟是在多大的範圍做一次性的行為會根據這個特性 (Attribute) 的使用位置而不同, 這部分文件是放在 [SetUpFixture Attribute 一節](https://github.com/nunit/docs/wiki/SetUpFixture-Attribute) 說明, 不過沒有所有情境的範例程式, 所以做了一些實驗來簡單紀錄一下更細節的部分.  

<!--more-->

### 使用方式與範圍
OneTimeSetUp 根據所使用的位置不同, 涵蓋的範圍會不同, 主要有三種: 
+ **類別層級 :**  
  該類別下加上 OneTimeSetUp 特性的方法只會執行一次.  
+ **命名空間層級 :**  
  該命名空間下加上 OneTimeSetUp 特性的方法只會執行一次, 且該類別上需要加 SetUpFixture 特性, __包含子命名空間__, 例如命名空間 `NS` 下的 OneTimeSetUp 在 `NS` 和 `NS.Sub` 命名空間總共只會執行一次.  
+ **組件層級 :**  
  若不在任何命名空間下, 則加上 OneTimeSetUp 特性的方法在整個組件中只會執行一次, 而該類別上也需要加 SetUpFixture 特性.  

### 範例程式碼
``` csharp
using NUnit.Framework;
using System.Diagnostics;

[SetUpFixture]
public class OneTimeSetupClass
{
    [OneTimeSetUp]
    public void OneTimeSetupMethod()
    {
        Debug.Print($"-- Assembly wide one-time setup.");
    }
}

namespace NS
{
    [SetUpFixture]
    public class OneTimeSetupClass
    {
        [OneTimeSetUp]
        public void OneTimeSetupMethod()
        {
            Debug.Print($"-- Namespace wide one-time setup.");
        }
    }

    [TestFixture]
    public class TestClass1
    {
        [OneTimeSetUp]
        public void OneTimeSetupMethod()
        {
            Debug.Print($"-- Class wide one-time setup.");
        }

        [Test]
        public void TestMethod1()
        {
            Debug.Print($"Executeing NS.TestClass1.TestMethod1()");
        }

        [Test]
        public void TestMethod2()
        {
            Debug.Print($"Executeing NS.TestClass1.TestMethod2()");
        }
    }

    [TestFixture]
    public class TestClass2
    {
        [OneTimeSetUp]
        public void OneTimeSetupMethod()
        {
            Debug.Print($"-- Class level one-time setup 2.");
        }

        [Test]
        public void TestMethod1()
        {
            Debug.Print($"Executeing NS.TestClass2.TestMethod1()");
        }

        [Test]
        public void TestMethod2()
        {
            Debug.Print($"Executeing NS.TestClass2.TestMethod2()");
        }
    }
}

namespace NS.Sub
{
    [TestFixture]
    public class TestClass1
    {
        [Test]
        public void TestMethod1()
        {
            Debug.Print($"Executeing NS.Sub.TestClass1.TestMethod1()");
        }

        [Test]
        public void TestMethod2()
        {
            Debug.Print($"Executeing NS.Sub.TestClass1.TestMethod2()");
        }
    }
}

namespace NotNS
{
    [TestFixture]
    public class TestClass1
    {
        [Test]
        public void TestMethod1()
        {
            Debug.Print($"Executeing NotNS.TestClass1.TestMethod1()");
        }

        [Test]
        public void TestMethod2()
        {
            Debug.Print($"Executeing NotNS.TestClass1.TestMethod2()");
        }
    }
}
```

以上是示範程式碼, 可以從下方的輸出訊息中看出一次性行為的範圍:  
```
-- Assembly wide one-time setup.
Executeing NotNS.TestClass1.TestMethod1()
Executeing NotNS.TestClass1.TestMethod2()
-- Namespace wide one-time setup.
Executeing NS.Sub.TestClass1.TestMethod1()
Executeing NS.Sub.TestClass1.TestMethod2()
-- Class wide one-time setup.
Executeing NS.TestClass1.TestMethod1()
Executeing NS.TestClass1.TestMethod2()
-- Class level one-time setup 2.
Executeing NS.TestClass2.TestMethod1()
Executeing NS.TestClass2.TestMethod2()
```

### 結論
因為 OneTimeSetUp 是根據所使用的位置不同而有不同的涵蓋範圍, 所以一開始有點不確定怎麼用, 就稍微寫一下簡單的筆記紀錄一下.  

### 參考
[SetUpFixture Attribute](https://github.com/nunit/docs/wiki/SetUpFixture-Attribute)  

