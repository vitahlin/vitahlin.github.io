---
title: Centos7安装gcc
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: install_gcc_in_centos7
date: 2017-03-06 14:54:33
updated: 2017-03-06 14:54:33
---

Centos 支持 `yum` 安装，安装命令为 
```c 
yum install xxx
```

按照这个思路，我们直接用命令安装 `yum install gcc`，安装完后可以查看 `gcc` 版本。但是查看 `g++` 版本却提示错误，命令未能识别。

**正确的安装命令应该是:**
```c 
yum install gcc-c++
```

安装完后可以用命令 `gcc -v` 和 `g++ -v` 查看版本。

参考链接
- [http://blog.csdn.net/robertkun/article/details/8466700](http://blog.csdn.net/robertkun/article/details/8466700)