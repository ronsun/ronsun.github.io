---
title: SymmetricAlgorithm 中 Key 和 KeySize 的依賴造成的少見問題
date: 2018-12-30 13:54:34
categories:
- C#
- .NET
tags:
---

之前在寫 AES 加解密用的工具方法的時候, 意外發現某個特殊的情境, 會導致解密時因為沒有 Key 而拋出例外 (但 Key 確實有設定).  

<!--more-->

### 問題解析

首先要來看一下示意的程式碼 

``` csharp
// Main logic
var aes = new AesCryptoServiceProvider()
{
    Mode = CipherMode.ECB,
    Key = GenerateKey(128),
    KeySize = 128,
}

string ciphertext = "jYCA05v0hL52/wT5WEJitQ==";
string plaintext = aes.Decrypt(ciphertext);
```

``` csharp
// extension method for decrypt
public static string Decrypt(this SymmetricAlgorithm symmetric, string ciphertext)
{
    string plaintext = string.Empty;

    MemoryStream ms = new MemoryStream();
    using (CryptoStream cs = new CryptoStream(ms, symmetric.CreateDecryptor(), CryptoStreamMode.Write))
    {
        var ciphertextBytes = Convert.FromBase64String(ciphertext);

        cs.Write(ciphertextBytes, 0, ciphertextBytes.Length);
        cs.FlushFinalBlock();
        plaintext = Encoding.UTF8.GetString(ms.ToArray());
    }
    return plaintext;
}
```

接著來分析一下錯誤內容  
**錯誤訊息**: System.Security.Cryptography.CryptographicException: 'Decrypting a value requires that a key be set on the algorithm object.'  
**堆疊追蹤最後一行**:   
`at System.Security.Cryptography.AesCryptoServiceProvider.CreateDecryptor()`

這看起來像是 Key 不存在, 但我們確實有指定 Key, 而用中斷點進去 `symmetric.CreateDecryptor()` 這一行的 `symmetric` 中卻發現 Key 的值已經不是一開始指定的值了.  

### 問題點說明

總之是耗了一點時間在排除問題, 最後發現是 `KeySize = 128` 這一行導致的, 因為**設定 KeySize 這件事本身會使得 Key 被釋放掉**.  

這個現象要看官方的原始碼才能知道, 從 [AesCryptoServiceProvider.cs](https://referencesource.microsoft.com/#System.Core/System/Security/Cryptography/AesCryptoServiceProvider.cs,123) 中可以看到 KeySize 的 setter 會將一開始設定的 Key 釋放掉, 導致 [AesCryptoServiceProvider.CreateDecryptor()](https://referencesource.microsoft.com/#System.Core/System/Security/Cryptography/AesCryptoServiceProvider.cs,144) 執行時發現 `m_key` 沒有值而拋錯. 

再更進一步去看父類別中 [SymmetricAlgorithm 中的 KeySize](https://referencesource.microsoft.com/#mscorlib/system/security/cryptography/symmetricalgorithm.cs,158) 就可以發現, 其實**不只 AES, 整個 SymmetricAlgorithm 的衍生類別都會在 KeySize 的 setter 中清掉現有的 Key** (除非有誰特別將這段邏輯覆寫掉).  

### 結論
相關的程式碼想看可以去上面那些連結裡面深入了解, 總之
+ Key 和 KeySize 之間是有依賴的, 指定 Key 之前如果先指定 KeySize 沒有意義, 因為指定 Key 的時候, KeySize 也會被改變, 所以下面範例**最後的 KeySize 是 256 而不是 128**. 
    ``` csharp
    new AesCryptoServiceProvider()
    {
        Mode = CipherMode.ECB,
        KeySize = 128,
        Key = GenerateKey(256),
    }
    ```
+ 同理, 如果指定了 Key, 就不可以在後面再指定 KeySize, 不然原本指定的 Key 會被清掉, 也就是本文遇到的問題. 
+ 需要指定 KeySize 的時機是, 加密時要指定 Key 的長度且不想自己產生 Key, 例如
    ``` csharp
    new AesCryptoServiceProvider()
    {
        Mode = CipherMode.ECB,
        KeySize = 128
    }
    ```
+ 相關問題, 網路上也有人討論, 附在下方參考中

其實這樣想起來也合理, Key 的值本身就可以算出 KeySize, 如果 KeySize 被指定時沒有清掉舊的 Key, 那就會可能發生 Key 的實際長度跟 KeySize 紀錄的長度不一樣的問題.  

然後 [Reference Source](https://referencesource.microsoft.com/) 和 [Source Browser - .net core](https://source.dot.net/) 真的是神物, 很多時候能靠這兩個東西了解細部的運作與問題發生的具體原因.  

### 參考
[RijndaelManaged - What does setting KeySize property do?](https://stackoverflow.com/questions/39275663/rijndaelmanaged-what-does-setting-keysize-property-do)  

---
