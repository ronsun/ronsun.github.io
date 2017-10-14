---
title: 將上游未合併的PR合併到自己的fork
date: 2017-10-14 21:31:31
category:
- Git
---

情境是這樣的, 我在github上fork了一個專案過來, 並且做了一些修改, 後來發現我的上游專案有幾個pull request剛好解決的我一直解決不了的問題, 該怎麼把那些變更同步到我的fork呢?

<!--more-->

先說一下這故事的人事時地物:  
+ 情境如上
+ JohnDao 他是pull request的作者
+ issue-fix-branch-1是這個變更所在的branch
+ AwesomeProject 是這個專案的名字  

**實作部分**  

先新增遠端儲存庫到要merge的來源專案  
```
git remote add JohnDao-Repo https://github.com/JohnDao/AwesomeProject.git  
```

然後把變更fetch下來  
```
git fetch JohnDao-Repo  
```

最後把變更merge進來, 然後push
```
git merge JohnDao-Repo/issue-fix-branch-1  
```

 **結論**  
這樣就能將別人(JohnDao)提交但是還沒被原作者合併回去的pull request先合併到自己這裡了。  

**參考資料**  
[How to apply unmerged upstream pull requests from other forks into my fork?](https://stackoverflow.com/questions/6022302/how-to-apply-unmerged-upstream-pull-requests-from-other-forks-into-my-fork)  
[git: how to merge a pull request into a fork?](https://stackoverflow.com/questions/36628859/git-how-to-merge-a-pull-request-into-a-fork)  
[What is “git remote add …” and “git push origin master”?](https://stackoverflow.com/questions/5617211/what-is-git-remote-add-and-git-push-origin-master) 
