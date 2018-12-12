---
title: Coding设置提交时不用重复输入账号密码
tags: Git
categories: Git
comments: true
copyright: true
abbrlink: not_repeat_input_password_in_coding_commit
date: 2017-08-07 16:37:33
updated: 2017-08-07 16:37:33
---

**当前系统：macOS 10.12.6**

当项目放在 `Coding` 上面时，使用 Http 协议进行推送每次都需要输入密码，这里记录可以免输入密码的设置。



```shell
# 没有就创建.git-credentials 
vi ~/.git-credentials
```

然后在 `.git-credentials` 里面输入内容：
```c
https://{username}:{passwd}@git.coding.net
```

`username` 是用户名，`passwd` 是密码。

然后执行配置 Git 命令存储认证命令：
```c
$git config --global credential.helper store
```

执行后在 `~/.gitconfig` 文件会多出下面配置项:
```c
credential.helper = store
```

这时候再提交就可以不用输入账户密码了。

参考链接：[https://coding.net/help/faq/git/git.html#push-](https://coding.net/help/faq/git/git.html#push-)