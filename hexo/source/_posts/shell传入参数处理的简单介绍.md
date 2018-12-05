---
title: shell传入参数处理的简单介绍
tags:
  - Linux&Unix
  - Shell
categories: Shell
comments: true
copyright: true
abbrlink: 3c449ea0
date: 2017-04-05 16:36:33
updated: 2017-04-05 16:36:33
---

如下一个命令：
```c 
./test.sh -f config.conf -v --prefix=/home
```

我们称`-f`为选项，它需要一个参数，即`config.conf`,`-v`也是一个选项，但它不需要参数。`--prefix`我们称之为一个长选项，即选项本身多于一个字符，它也需要一个参数，用等号连接，当然等号不是必须的，`/home`可以直接写在`--prefix`后面，即`--prefix/home`。

<!--more-->

以上述的命令为例：

*    $0 :  `./test.sh`，即命令本身，相当于C/C++中的 `argv[0]`
*    $1 :  `-f`，第一个参数
*    $2 :  `config.conf`，第二个参数
*    $3, $4 ... ：类推
*    $# :  参数的个数，不包括命令本身，上例中 `$#`为 **4**
*    $@ :  参数本身的列表，也不包括命令本身，如上例为 `-f`、`config.conf`、`-v`、`--prefix=/home`
*    $\* :  和 `$@` 相同，但 `$*` 和 `$@` (加引号)并不同， `$*` 将所有的参数解释成一个字符串，而 `$@` 是一个参数数组。


参考链接
- [http://blog.csdn.net/qzwujiaying/article/details/6371246](http://blog.csdn.net/qzwujiaying/article/details/6371246)