---
title: HttpRequest.QueryString 的自動 UrlDecode 問題
date: 2017-08-13 01:33:03
categories:
- C#
- .NET
---

HttpRequest.QueryString["q"] 在用來取得query string的值是很方便, 但是會自動將query string先做過UrlDecode, 這在query string 有某些特殊字元的時候產生問題

<!-- more -->

直接上Demo

``` csharp
        public ActionResult Index()
        {
            //sample query string (plain text): q=aaa$bbb+ccc

            //q = "aaa$bbb ccc"
            string q = this.Request.QueryString["q"];


            // solution 1: encode after get query string
            //             useful if query string not contain other characters can be encode
            //q = "aaa%24bbb+ccc" in this case
            string q2 = HttpUtility.UrlEncode(this.Request.QueryString["q"]);

            // solution 2: split string to get value
            //             if query string contain charactors can be encode, this is safety way
            List<string> queryStrings = this.Request.Url.Query.Replace("?", "").Split('&').ToList();
            //q3 = "aaa$bbb+ccc"
            string q3 = queryStrings.Where(r => r.Split('=')[0] == "q").Select(r => r.Split('=')[1]).FirstOrDefault();

            return View();
        }
```











