---
title: 'SSH登陆错误 WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!'
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: 3615d315
date: 2017-03-01 21:52:33
updated: 2017-03-01 21:52:33
---

### 问题描述

macOS上通过 `ssh` 登陆时，出现登陆错误信息：
```c 
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:xdN/+gIKPhQqbPtryxRiK3grcpTZ5OE0oiRX5JnopWg.
Please contact your system administrator.
Add correct host key in /Users/vitah/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/vitah/.ssh/known_hosts:7
ECDSA host key for 120.26.68.150 has changed and you have requested strict checking.
Host key verification failed.
```

### 解决
`known_hosts`文件位置：
```c 
vi ~/.ssh/known_hosts
```

用 Vim 打开 `known_hosts` 文件，删除 `known_hosts` 文件中**对应 IP 的那一行**。

### 参考链接
- [http://www.cnblogs.com/dongzhiquan/archive/2012/10/09/2717439.html](http://www.cnblogs.com/dongzhiquan/archive/2012/10/09/2717439.html)