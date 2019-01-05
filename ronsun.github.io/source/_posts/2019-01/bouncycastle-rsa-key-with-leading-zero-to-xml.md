---
title: 將 RSA 金鑰轉成 XML 格式的少見問題
date: 2019-01-05 20:20:17
categories:
- C#
- Packages
tags:
---

.NET 內建的 RSA 加解密相關元件可以用讀取憑證檔或是 XML 格式的金鑰的方式來初始化, 之前的專案都是用讀取 XML 的方式來操作, 讓管理者能夠方便的從後台來管理金鑰.  

而在 PEM 檔中, 金鑰的格式是 base64 字串, 這用 JAVA 是可以正常讀取的, 但卻不是 .NET 接受的 XML 格式, 因此用了一個第三方套件 [Bouncy Castle](https://github.com/bcgit/bc-csharp) 來幫忙轉換, 但是在一個少見的情境下 (其實也沒有多少見, 我用 OpenSSL 隨機生 500 組, 就有 10 組觸發), 轉換出來的 XML 是錯誤的, 且只有私鑰有發生過.  

<!--more-->

### 問題
如果有在網路上找過金鑰轉換的方式的話, 應該不難找到下面段轉換私鑰的程式碼, 透過 `RSAPrivateKeyJava2DotNet(string privateKey)` 方法來將 base64 格式的私鑰轉換成 XML 格式.  

``` csharp
public string SignDataByPrivateKey(string data, string key)
{
    string xmlData = RSAPrivateKeyJava2DotNet(key);
    RSACryptoServiceProvider privateKey = new RSACryptoServiceProvider();
    privateKey.FromXmlString(xmlData);
    var signBytes = privateKey.SignData(Encoding.UTF8.GetBytes(data), "sha1");
    return Convert.ToBase64String(signBytes);
}

public string RSAPrivateKeyJava2DotNet(string privateKey)
{
    RsaPrivateCrtKeyParameters privateKeyParam = (RsaPrivateCrtKeyParameters)PrivateKeyFactory.CreateKey(Convert.FromBase64String(privateKey));

    return string.Format(
        "<RSAKeyValue><Modulus>{0}</Modulus><Exponent>{1}</Exponent><P>{2}</P><Q>{3}</Q><DP>{4}</DP><DQ>{5}</DQ><InverseQ>{6}</InverseQ><D>{7}</D></RSAKeyValue>",
        Convert.ToBase64String(privateKeyParam.Modulus.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.PublicExponent.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.P.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.Q.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.DP.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.DQ.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.QInv.ToByteArrayUnsigned()),
        Convert.ToBase64String(privateKeyParam.Exponent.ToByteArrayUnsigned())
    );
}
```

這段程式碼一般情境下沒有問題, 但是如果用下面的測試私鑰來試, 就會在 `privateKey.FromXmlString(xmlData);` 這一行拋出 `CryptographicException` 且錯誤資訊只有短短的 "Bad Data.\r\n".  

**Private Key** :  
`MIICdwIBADANBgkqhkiG9w0BAQEFAASCAmEwggJdAgEAAoGBAOEZbmixVVPO6Z96
sGqb8G3gBOgBuLc6o673GyxLZPXPk6VaBF+2LJ+WnIbuYu3iDE2pR1SHeA09BCyz
YJnJTYl77vqzRqyzf9lWHGJDkwpEnftlDfeIc5ICfWpu3bHuBuzmtIfqrErCbaJJ
3HkhOsuJ3oL8dAYQN5RTt66IpSn9AgMBAAECgYBEOuotj7sWeTx1W8IHvpbFJ0c1
b/gmif69dSdmaMAEhlPxpfR3cofaI9P0TmPsST2DeNEnPRzVnm4agpDAbLU0ana1
I8pfjRq3xrlTJkDjWKOyaFF2afo2y1pyIFsm1G5wVEFwKWSNUo/1Jy61H4snYxMk
/SCZqAYxO3S12jAFHQJBAP6Cv1MV8HcBMuDgyl9Obq/tU6r9zrGOmR1bWBA61DVw
jcWgMSpo8dkBAs0Hd5SfWNNzUIVH2gryAcy5Skyq8mMCQQDiaqBM2nCLfQ3EaLzy
e1tYZCuZYiZl60OX9nS9FNLFwSI6sFsy/+aGg1ivSVHvXpi9N1oaNgvSI13z9DT0
B/AfAkAArfqyzxkwSCmJnjAMJxp2j8ysZTcbFEVmZasLiAyvA9jtEStwcI1Mxgrq
3z07gV1sWx9466MyakkE8e233LD/AkEA244n+b6NCkZu1jn2l3CVaIZiXO93areT
qUV9eGk75jXdemnPVgoeQewWUIvZ3zOtCzcksWwdVF2lWs5Bly4nYwJBAK+niJt3
7jWf13muppL1aRN/i1otm3yNZuNGhrCZ/nEfehhBtege3IHn337fuyLhlXMc37OY
k1iKRpcW8kn4LFQ=`  

### 原因
當初也是找了很久, 最後發現 `privateKeyParam.DP.ToByteArrayUnsigned()` 中的 DP 只有 63 bytes, 少了一個位元組, 而造成這個現象的原因是因為 `privateKeyParam.DP` 是數值型態, `Org.BouncyCastle.Math.BigInteger` 在初始化的時候, 跳過了開頭的所有 0, 而且這個問題在 DP , DQ , Exponent , QInv 這四段上面都會發生.  

> 根據[官方JIRA 上的單子 BMA-92](http://www.bouncycastle.org/jira/browse/BMA-92?jql=project%20%3D%20BMA%20AND%20fixVersion%20%3D%201.8.0) 可以看出是在 `Org.BouncyCastle.Math.BigInteger` 初始化過程造成的, 從原始碼中看起來也很合理(不過我只是[看看原始碼推測的](https://github.com/bcgit/bc-csharp/blob/f18a2dbbc2c1b4277e24a2e51f09cac02eedf1f5/crypto/src/math/BigInteger.cs), 沒有直接拉下來驗證, 不保證真的是):  
> 從 public BigInteger(byte[] bytes) : this(bytes, 0, bytes.Length)  
> => public BigInteger(byte[] bytes, int offset, int length)  
> => private static int[] MakeMagnitude(byte[]	bytes, int offset, int length)  
> 中的 firstSignificant 這個 flag 以第一個非 0 位置為起點

而用這個發現來推論, 也可以理解為什麼公鑰不會有這問題, 因為公鑰只有 Modulus 和 Exponent 這兩段.  

### 解決方式
其實這個轉換過程 Bouncy Castle 有提供更簡便的方式, 不需要自己串, 將 `RSAPrivateKeyJava2DotNet(string privateKey)` 改成像下面這樣(公鑰的部分也一併改了):  

``` csharp
private static string RSAPrivateKeyJava2DotNet(string privateKey)
{
    RsaPrivateCrtKeyParameters privateKeyParam = (RsaPrivateCrtKeyParameters)PrivateKeyFactory.CreateKey(Convert.FromBase64String(privateKey));
    return DotNetUtilities.ToRSA(privateKeyParam).ToXmlString(true);
}

public static string RSAPublicKeyJava2DotNet(string publicKey)
{
    RsaKeyParameters publicKeyParam = (RsaKeyParameters)PublicKeyFactory.CreateKey(Convert.FromBase64String(publicKey));
    return DotNetUtilities.ToRSA(publicKeyParam).ToXmlString(false);
}
```

如果有追進去[原始碼](https://github.com/bcgit/bc-csharp/blob/f18a2dbbc2c1b4277e24a2e51f09cac02eedf1f5/crypto/src/security/DotNetUtilities.cs)看可以發現, 他在要轉換成 XML 字串前, 將開頭有 0 被忽略的部分再補回來, 實測後也確實可以避免原本的問題, 他的 call stack 有點長, 這裡直接列出最關鍵的地方如下, 就是 `ConvertRSAParametersField` 這個方法才避免掉這個問題的:  

``` csharp
public static RSAParameters ToRSAParameters(RsaPrivateKeyStructure privKey)
{
    RSAParameters rp = new RSAParameters();
    rp.Modulus = privKey.Modulus.ToByteArrayUnsigned();
    rp.Exponent = privKey.PublicExponent.ToByteArrayUnsigned();
    rp.P = privKey.Prime1.ToByteArrayUnsigned();
    rp.Q = privKey.Prime2.ToByteArrayUnsigned();
    rp.D = ConvertRSAParametersField(privKey.PrivateExponent, rp.Modulus.Length);
    rp.DP = ConvertRSAParametersField(privKey.Exponent1, rp.P.Length);
    rp.DQ = ConvertRSAParametersField(privKey.Exponent2, rp.Q.Length);
    rp.InverseQ = ConvertRSAParametersField(privKey.Coefficient, rp.Q.Length);
    return rp;
}

private static byte[] ConvertRSAParametersField(BigInteger n, int size)
{
    byte[] bs = n.ToByteArrayUnsigned();

    if (bs.Length == size)
        return bs;

    if (bs.Length > size)
        throw new ArgumentException("Specified size too small", "size");

    byte[] padded = new byte[size];
    Array.Copy(bs, 0, padded, size - bs.Length, bs.Length);
    return padded;
}
```

### 案外案 - 套件版本
**但是**  
**但是**  
**但是**  

**剛剛的解決方式只在 1.8.0 之後才有效, 1.7.x 以前的版本即使這樣使用還是會出錯.**  

這部分從上面有出現過一個 [JIRA 的列表](http://www.bouncycastle.org/jira/browse/BMA-92?jql=project%20%3D%20BMA%20AND%20fixVersion%20%3D%201.8.0)就可以看到這個問題是 1.8.0 才修復的, 但是 GitHub 的時候已經是 1.8.0 版了, 所以如果真的想比對舊版的原始碼的話, 就要去[官網](https://www.bouncycastle.org/csharp/index.html)考古了.  

### 結論 
其實如果直接讀憑證檔, 就不會有這個問題, 但是專案特性的關係, 這個金鑰會常常新增或更新, 為了不要每次增加或修改都要重新佈署, 只能花時間下去找原因跟解法了.  

當時遇到這個問題的時候真的是很難過, 因為錯誤訊息超級少, 想找 google 都不知道關鍵字怎麼下, 還好弄了兩三天之後意外發現資料長度不對才有突破口, 這個專案中加解密的部分是存在好幾年的共用方法, 一直沒人遇到過這個情境, 既然被我踩到了也算是得到一個難得的經驗吧.  

另外, 既然套件有封裝好這麼方便的方法, 之後有需要寫類似的方法時, 就不用再用以前流傳的那種方式了, 直接用解法中的做法就好, 這也是在網路上搜尋解決方案的時候要注意的, 多找幾種解決方案比較過再決定, 會更有機會找到比較適合的解決方案.  

### 參考
[RSA Key Formats - Key File Encoding](https://www.cryptosys.net/pki/rsakeyformats.html)  
[C#和JAVA的RSA密钥、公钥转换](https://blog.csdn.net/yupu56/article/details/72624229)  
[JIRA of Bouncy Castle](http://www.bouncycastle.org/jira/browse/BMA-92?jql=project%20%3D%20BMA%20AND%20fixVersion%20%3D%201.8.0)  

---
