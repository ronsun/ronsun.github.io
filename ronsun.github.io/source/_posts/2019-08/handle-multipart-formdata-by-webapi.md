---
title: 在 WebAPI 中處理 multipart/form-data 資料
date: 2019-08-26 00:13:21
categories:
- C#
- .NET
tags:
---

在跟外部廠商介接的過程發現少數廠商在發送到我們的 WebAPI 的 request 是 `multipart/form-data` 的, 以前舊的老專案中要處理這種資料還要透過第三方套件, 後來在遷移到 .NET 的過程中發現,用 .NET 處理這類型的資料是可以不用第三方套件的, 所以紀錄一下碰到的幾種情境的處理方式.  

<!--more-->

這邊除了用 `MultipartFormDataContent` 和 `HttpClient` 來模擬客戶端呼叫我們的 WebAPI 之外, 也會用 postman 模擬 (比較方便不用寫程式), 而用 postman 模擬要注意不要額外將 `content-type` 設成 `multipart/form-data`, 理由[這篇討論的回應](https://stackoverflow.com/questions/24503961/invalid-httpcontent-instance-provided-it-does-not-have-a-multipart-content/43868741#43868741)中有提到.  

### 包含 FileName
客戶端呼叫 API 的程式碼約略如下: 
``` csharp
using (var content = new MultipartFormDataContent())
{
	content.Add(new StringContent("key1"), "value1", "file1.txt");
	content.Add(new StringContent("key2"), "value2", "file2.txt");

	var req = new HttpClient();
	req.PostAsync("http://localhost:50640/api/multipart", content).Wait();
}
```

用 postman 模擬的話會是這樣:  
{% asset_img multipart-with-file.png %}   


而 body 的內容如下: 
```
--29874ecd-2a56-406a-a2c0-18eff0d3ae85
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name=value1; filename=file1.txt; filename*=utf-8''file1.txt

key1
--29874ecd-2a56-406a-a2c0-18eff0d3ae85
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name=value2; filename=file2.txt; filename*=utf-8''file2.txt

key2
--29874ecd-2a56-406a-a2c0-18eff0d3ae85--
```
#### 從 ApiController.Request 取得資料
這個情境下 `ApiController.Request.Content.IsMimeMultipartContent()` 會回傳 `true`, 可用這個方法來作為判斷基準.  

``` csharp
private async void GetMultipartFromHttpRequestMessage()
{
	Debug.WriteLine("===== Get from HttpRequestMessage (multipart) =====");

	var multi = await Request.Content.ReadAsMultipartAsync();
	Debug.WriteLine("===== Get from file =====");
	foreach (var content in multi.Contents)
	{
		string key = content.Headers.ContentDisposition.Name;
		string value = string.Empty;
		using (Stream s = content.ReadAsStreamAsync().Result)
		using (StreamReader sr = new StreamReader(s))
		{
			s.Seek(0, SeekOrigin.Begin);
			value = sr.ReadToEnd();
			Debug.WriteLine($"key: {key}, value: {value}");
		}
	}
}
```
### 不含 filename
客戶端呼叫 API 的程式碼約略如下, 差別在於 `contnet.Add(...)` 沒有帶第三個參數: 
``` csharp
using (var content = new MultipartFormDataContent())
{
	content.Add(new StringContent("key1"), "value1");
	content.Add(new StringContent("key2"), "value2");

	var req = new HttpClient();
	req.PostAsync("http://localhost:50640/api/multipart", content).Wait();
}
```

用 postman 模擬的話會是這樣, 差別在不是用檔案上傳:  
{% asset_img multipart.png %}   

而 body 的內容如下: 
```
--9b753acb-2b3a-4214-bfc4-47b25799436f
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name=value1

key1
--9b753acb-2b3a-4214-bfc4-47b25799436f
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name=value2

key2
--9b753acb-2b3a-4214-bfc4-47b25799436f--
```

雖然跟上一個例子比起來, 似乎只少了 Content-Disposition 中的 filename 一段, 但這會讓我們在取資料上變得更方便.  

#### 從 ApiController.Request 取得資料
在不含 filename 的情境下,  `ApiController.Request.Content.IsMimeMultipartContent()` 依然會是 `true`, 所以上一個情境的解法依然適用.  

#### 從 HttpContext.Current.Request 取得資料
在不含 filename 的情境下, 操作上可以視同一般的 form post 來取得資料, 不用做額外的處理, 如下:
``` csharp
private void GetFormFromHttpRequest()
{
	Debug.WriteLine("===== Get from HttpRequestMessage (form) =====");
	var formData = HttpContext.Current.Request.Form;
	foreach (var key in formData.AllKeys)
	{
		string value = formData[key];
		Debug.WriteLine($"key: {key}, value: {value}");
	}
}
```

有趣的地方是雖然操作上跟一般 form post 一樣, 但是如果試圖透過 `ApiController.Request.Content.ReadAsFormDataAsync()` 取得資料是會拋出例外, 且
`ApiController.Request.Content.IsFormData()` 的回傳值是 `false`.  

### 結論
目前因為確定合作廠商使用 `multipart/form-data` 的情境都不會帶 filename, 所以是視同一般 form post 直接從`HttpContext.Current.Request` 中取得資料, 省去額外的判斷跟處理.  

但嚴格來說, 不管是不是有 filename 這段, 都應該用 `ApiController.Request.Content.ReadAsMultipartAsync()` 來處理比較恰當, 目前的做法比較像是因為剛好可以相容又可以省事, 就這樣用了.  

### 參考
[24503961](https://stackoverflow.com/questions/24503961/invalid-httpcontent-instance-provided-it-does-not-have-a-multipart-content/43868741#43868741)  
[How to get string representation of a MultipartFormDataContent](https://stackoverflow.com/questions/41713215/how-to-get-string-representation-of-a-multipartformdatacontent)  