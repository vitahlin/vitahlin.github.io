---
title: CentOS 6.8 编译安装 Git
tags: Git
categories: Git
comments: true
copyright: true
abbrlink: install-git-in-centos6
date: 2017-08-10 10:21:33
updated: 2017-08-10 10:21:33
---

在 `CentOS 6`上通过`yum` 安装的 git 版本只有 1.7，在使用时经常会出现错误，所以这里介绍通过编译安装 Git 2.11。

### 安装依赖包

```c
yum install curl-devel expat-devel gettext-devel \
  openssl-devel zlib-devel
```

<!--more-->

### 下载

```c
cd /usr/local/src
wget https://www.kernel.org/pub/software/scm/git/git-2.11.1.tar.gz
```

### 解压

```c
tar zxvf git-2.11.1.tar.gz
```

### 编译

```c
cd /usr/local/src/git-2.11.1
make prefix=/usr/local all
sudo make prefix=/usr/local install
```

之后就可以使用 `Git` 了。

### 参考链接
- [https://git-scm.com/book/zh/v1/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git](https://git-scm.com/book/zh/v1/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git)
