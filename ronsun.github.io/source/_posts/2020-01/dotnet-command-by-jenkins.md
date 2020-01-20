---
title: 透過 Jenkins 開關 dotnet core 應用程式
date: 2020-01-20 23:17:26
categories:
- C#
- .NET Core
tags:
---

前陣子做了一個 dotnet core 的新專案, 上線前需要先跟 Jenkins 佈署流程整合, 本來很單純的想說用 `dotnet MyCore.dll` 指令開啟應用程式, 至於停止的時候就暴力用 `kill` 指令來關閉 (當然這種暴力解只是在實驗階段用來建立信心的, 接下來還是要找看有沒有其他更好的做法).  

沒想到連信心都建立不起來, 透過 Jenkin 用  `kill` 指令時, 會出現錯誤 (沒有把錯誤細節記錄下來, 反正用 `kill` 也太暴力, 終究還是要換個方法的, 所以就果斷放棄去找尋新方法了).  

<!--more-->

### 使用 dotnet 指令
方便歸方便, 但如同一開始說的, 跟 Jenkins 整合的時候會有問題, 這裡就是順便紀錄一下指令而已, 沒什麼特別的.  

#### 開啟應用程式
`dotnet MyCore.dll > /dev/null 2>&1 &`

#### 停止應用程式
`kill -9 11905`

> 11905 是 pid

### 用 systemctl 指令管理

#### 建立服務
**新增 MyCore.service**  
`vim /etc/systemd/system/MyCore.service`

**編輯內容**  

```
[Unit]
Description=This is MyCore

[Service]
WorkingDirectory=/usr/dotnet/MyCore/
ExecStart=/usr/bin/dotnet /usr/dotnet/MyCore/MyCore.dll
Environment=ASPNETCORE_ENVIRONMENT=Development

[Install]
WantedBy=multi-user.target
```

> 1. 這裡要注意的是環境變數要設定在服務檔裡面才有效.  
> 1. 上面服務檔的範例只是極簡版, 能動而已, [官方範例參考這裡](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-2.1#create-the-service-file)  

**啟用**  
用 `systemctl` 開關應用程式.  

```
systemctl start MyCore  
systemctl stop MyCore  
```

### 結論
如果會用 linux 的話其實不用特別紀錄, 只要知道用 `systemctl` 取代 `dotnet` 來開關應用程式就好, 但是我很不會用 linux, 所以還是要備忘一下, 免得下次遇到一樣的問題要 google 連關鍵字都不會下.  

### 參考
[Manage Kestrel process with systemd](https://kimsereyblog.blogspot.com/2018/05/manage-kestrel-process-with-systemd.html)  
[Host ASP.NET Core on Linux with Nginx](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-2.1#create-the-service-file)  
[How to run ASP.NET Core as a service on Linux (RHEL)](https://swimburger.net/blog/dotnet/how-to-run-aspnet-core-as-a-service-on-linux)  
