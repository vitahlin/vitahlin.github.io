---
title: 撤销git commit
tags: Git 
categories: Git
comments: true
copyright: true
abbrlink: reset_git_commit
date: 2018-12-11 10:39:33
updated: 2018-12-11 10:39:33
---

在`git push`的时候，有时候我们想撤销`git commit`的内容。方法如下：

1. 找到之前提交的`git commit`的`ID`

通过执行命令`git log`可以查看：
```c
commit 309313ac0d0144319deea62232fb33450d6c1481 (HEAD -> master)
Author: Vitah <linw1225@gmail.com>
Date:   Wed Oct 31 16:07:44 2018 +0800

    Test：测试commit取消

commit 4a4f71857d11a70c3e4c5b6afae319820547daf7 (origin/master, origin/HEAD)
Author: Vitah <linw1225@gmail.com>
Date:   Wed Oct 31 15:55:40 2018 +0800

    Refactor：重构错误码获取相关代码

commit daeef72dae8feda10f437a528bc01696e49fb960
Author: Vitah <linw1225@gmail.com>
Date:   Wed Oct 31 15:41:22 2018 +0800

    Refactor：错误码单独封装
```

如上，上一个`git commit`的 ID 即是：`4a4f71857d11a70c3e4c5b6afae319820547daf7`

2. 执行撤销操作

执行撤销操作有两种方式：
- `git reset --hard id` 完成撤销，同时将代码恢复到之前commit的版本，即同时删除你的代码更改
- `git reset id` 完成 commit 命令的撤销，但是不对代码修改进行撤销，可以直接通过 git commit 重新提交对本地代码的修改