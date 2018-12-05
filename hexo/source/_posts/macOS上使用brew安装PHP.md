---
title: macOS上使用brew安装PHP
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: install_php_via_brew
date: 2016-12-15 14:52:33
updated: 2016-12-15 14:52:33
---

**系统版本是macOS 10.12.1**

### brew安装PHP

安装前brew要升级到最新的版本，不然容易出问题。
然后用如下命令安装：
```c
brew tap josegonzalez/php #安装扩展<gihhub_user/repo>
brew tap #查看安装的扩展列表
brew install php55 #安装php5.5
```
<!--more-->

由于Mac本身自带PHP，因此需要添加系统环境便利PATH来替代自带PHP版本：
```c
sudo vim ~/.bash_profile
// 输入
export PATH="$(brew --prefix php55)/bin:$PATH"
export PATH="$(brew --prefix php55)/sbin:$PATH"

source ~/.bash_profile
```

接下来可以测试：
```c
#brew安装的php 他在/usr/local/opt/php55/bin/php
php -v
#Mac自带的PHP
/usr/bin/php -v
#brew安装的php-fpm 他在/usr/local/opt/php55/sbin/php-fpm
php-fpm -v
#Mac自带的php-fpm
/usr/sbin/php-fpm -v
```

### 启动Apache
Mac上已经自带Apache，可以不用安装，直接启动即可。
```c
sudo apachectl start // 启动
sudo apachectl stop // 停止
sudo apachectl restart // 重启
```
运行启动命令启动Apache，然后在浏览器里输入`http://localhost/`，显示**`It works! `**即启动成功。