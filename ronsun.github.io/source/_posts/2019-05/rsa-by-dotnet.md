---
title: RSA 加解密與簽名
date: 2019-05-18 01:36:23
categories:
- C#
- .NET
tags:
---

.NET 提供的 RSACryptoServiceProvider 用來做 RSA 加解密與驗簽非常方便, 相關的網路資源也很多, 本來是沒什麼好寫的畢竟隨便 google 一下都有, 但是最近發現不知道為什麼用一次查一次然後忘一次, 想想還是稍微紀錄一下用法方便以後速查好了.  

<!--more-->

### 加解密
#### 使用 XML 格式的公私鑰
沒什麼特別的, 就是用 RSACryptoServiceProvider 加解密.  

``` csharp
public string RSAEncryptFromXMLKey(string publicKey, string plaintext)
{
	RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
	rsa.FromXmlString(publicKey);

	var plaintextBytes = Encoding.UTF8.GetBytes(plaintext);
	var ciphertextBytes = rsa.Encrypt(plaintextBytes, false);
	var ciphertext = Convert.ToBase64String(ciphertextBytes);
	return ciphertext;
}

public string RSADecryptFromXMLKey(string privateKey, string ciphertext)
{
	RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
	rsa.FromXmlString(privateKey);

	var ciphertextBytes = Convert.FromBase64String(ciphertext);
	var plaintextBytes = rsa.Decrypt(ciphertextBytes, false);
	var plaintext = Encoding.UTF8.GetString(plaintextBytes);
	return plaintext;
}
```

#### 載入 pfx 檔
``` csharp
public string RSAEncryptFromPFX(string plaintext)
{
	X509Certificate2 certificate = new X509Certificate2("path/to/certificate.pfx", "password");
	var rsa = (RSACryptoServiceProvider)certificate.PublicKey.Key;

	var plaintextBytes = Encoding.UTF8.GetBytes(plaintext);
	var ciphertextBytes = rsa.Encrypt(plaintextBytes, false);
	var ciphertext = Convert.ToBase64String(ciphertextBytes);
	return ciphertext;
}

public string RSADecryptFromPFX(string ciphertext)
{
	X509Certificate2 certificate = new X509Certificate2("path/to/certificate.pfx", "password");
	var rsa = (RSACryptoServiceProvider)certificate.PrivateKey;

	var ciphertextBytes = Convert.FromBase64String(ciphertext);
	var plaintextBytes = rsa.Decrypt(ciphertextBytes, false);
	var plaintext = Encoding.UTF8.GetString(plaintextBytes);
	return plaintext;
}
```

> 註:  
> 1. 方便示範所以把一些參數與物件都寫死在方法中, 實務上該抽參數或是抽離出 X509Certificate2 的操作等都要再考慮

### 簽名與驗簽
#### 使用 XML 格式的公私鑰
``` csharp
public string RSASignFromXMLKey(string privateKey, string plaintext)
{
	RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
	rsa.FromXmlString(privateKey);

	var plaintextBytes = Encoding.UTF8.GetBytes(plaintext);
	var signBytes = rsa.SignData(plaintextBytes, new SHA1CryptoServiceProvider());
	var sign = Convert.ToBase64String(signBytes);
	return sign;
}

public bool RSAVerifyFromXMLKey(string publicKey, string sign, string data)
{
	RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
	rsa.FromXmlString(publicKey);

	var signBytes = Convert.FromBase64String(sign);
	var dataBytes = Encoding.UTF8.GetBytes(data);
	bool isValid = rsa.VerifyData(dataBytes, new SHA1CryptoServiceProvider(), signBytes);
	return isValid;
}
```

這也沒什麼難的, 不過 VerifyData 這個方法第一個參數是明文, 第三個參數是簽名後的密文比較容易弄反.  

另外一個有趣的地方是 VerifyData 的第二個參數和 SignData 的第二個參數型別是 `object`, 這意味著他可以接受任意型別的輸入, 例如: `new SHA1CryptoServiceProvider()` 或 `"sha1"`(不分大小寫的字串), 在使用上我會傾向在封裝 wrapper 時刻意限縮個參數的的型別, 否則呼叫端會很難預期傳入的值是不是正確的, 例如以上例為延伸, 下面的每一行程式碼如果沒經過實測很道運作結果是否一樣(而事實上結果是一樣的).  

``` csharp
var signBytes = rsa.SignData(plaintextBytes, "sha1");
var signBytes = rsa.SignData(plaintextBytes, "SHA1");
var signBytes = rsa.SignData(plaintextBytes, "SHa1");
var signBytes = rsa.SignData(plaintextBytes, SHA1.Create());
var signBytes = rsa.SignData(plaintextBytes, new SHA1Managed());
var signBytes = rsa.SignData(plaintextBytes, new SHA1CryptoServiceProvider());
var signBytes = rsa.SignData(plaintextBytes, new SHA1Cng());
var signBytes = rsa.SignData(plaintextBytes, typeof(SHA1));
var signBytes = rsa.SignData(plaintextBytes, typeof(SHA1Managed));
// etc...
```

#### 載入 pfx 檔
用私鑰簽名時載入 pfx 擋的方式跟加解密時一樣, 簽名與驗簽的相關方法都跟上例一樣, 就不重複貼程式碼了.  

### 結論
目前只知道 XML 格式的公私鑰和 PFX 格式的憑證擋在 .NET 下是最方便操作的, 如果還要讀取自其他格式的檔案或公私鑰內容(例如 PEM), 大多要靠其他套件才比較好做, 例如: [Bouncy Castle](https://github.com/bcgit/bc-csharp).  

### 參考
