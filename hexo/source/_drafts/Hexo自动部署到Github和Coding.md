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

这里使用`docker-compose`，具体`docker-compose`配置代码可以参照：[https://github.com/vitahlin/vitahlin.github.io/blob/develop/docker/docker-compose.yml](https://github.com/vitahlin/vitahlin.github.io/blob/develop/docker/docker-compose.yml)
`hexo`镜像相关配置在`docker/hexo/Dockerfile`里面，查阅相关代码即可，这里不再赘述。

#### 
