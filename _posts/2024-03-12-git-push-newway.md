---
layout: post
title: github 推送仓库新方式
tags: git使用
---

新方式说明，使用token。

##### 1、设置

```
git remote set-url origin https://<token>@<仓库>
仓库eg:github.com/wbxx
```

##### 2、前提需要add remote链接

```
git remote remove origin
git remote add origin <仓库>
```

##### 3、推送

```
git branch
git push -u origin master
```

查看链接wbxx.github.io即可。