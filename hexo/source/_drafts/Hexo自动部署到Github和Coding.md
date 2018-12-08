---
title: Hexo自动部署到Github和Coding
tags:
  - Blog
  - Hexo
categories: Blog
comments: true
copyright: true
abbrlink: auto_deploy_hexo_to_github_and_coding
date: 2017-08-03 20:29:33
updated: 2017-08-03 20:29:33
---

### 概述

Hexo 是一个快速、简洁且高效的博客框架。
本文介绍如何使用`docker`管理`hexo`环境以及用`Travis CI`实现自动部署到Github Page和Coding Page上。


#### 使用`docker`

使用`docker`可以不需要每次重新安装`hexo`环境。

我们用`docker-compose`实现`hexo`的镜像，这样本地只需要`docker`即可，具体配置及其使用方法可以参考这里：[https://github.com/vitahlin/Dockers/tree/master/hexo](https://github.com/vitahlin/Dockers/tree/master/hexo)


#### `hexo`使用

在容器内执行对应的`hexo`命令即可。