---
title: macOS上brew安装Mysql
tags: Database
categories: Database
comments: true
copyright: true
abbrlink: install_mysql_via_homebrew
date: 2017-08-11 18:28:33
updated: 2017-08-11 18:28:33
---

### 安装

执行命令：
```c
brew install mysql
```

运行结果：
```c
We've installed your MySQL database without a root password. To secure it run:
    mysql_secure_installation
MySQL is configured to only allow connections from localhost by default
To connect run:
    mysql -uroot
To have launchd start mysql now and restart at login:
  brew services start mysql
Or, if you don't want/need a background service you can just run:
  mysql.server start
[proxychains] DLL init: proxychains-ng 4.12
==> Summary
🍺 /usr/local/Cellar/mysql/5.7.19: 322 files, 233MB
```

<!--more-->

### 启动服务

**运行后，如果使用 `LaunchRocket` 来启动 `Mysql` 服务是不生效的，启动和关闭 `Mysql` 服务只能采用命令行。**


#### 启动

```c
mysql.server start
```

#### 关闭

```c
mysql.server stop
```

#### 重启

```c
mysql.server restart
```

#### 查看是否在运行

```c
mysql.server status
```

### 设置

先采用命令行启动 mysql 服务后，在执行相关安全设置，输入命令：
```c
mysql_secure_installation
```