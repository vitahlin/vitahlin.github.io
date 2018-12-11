---
title: Centos7通过nvm安装Node.js
tags:
  - Linux&Unix
  - Node.js
categories:
  - Node.js
comments: true
abbrlink: install_nodejs_with_nvm_in_centos7
date: 2016-12-06 16:29:33
updated: 2016-12-06 16:29:33
copyright: true
---

### 克隆最新源码
```c
git clone https://github.com/creationix/nvm.git /usr/local/nvm
```

### 安装nvm
```c
source usr/local/nvm/install.sh
```

<!--more-->

### 列出Node.js所有远程版本
```c
nvm ls-remote
```

### 安装特定版本
如：
```c
nvm install v4.0.0
```

### 参考链接
- [http://www.smartshare.top/blog/%E4%BD%BF%E7%94%A8nvm%E5%AE%89%E8%A3%85nodejs/](http://www.smartshare.top/blog/%E4%BD%BF%E7%94%A8nvm%E5%AE%89%E8%A3%85nodejs/)
