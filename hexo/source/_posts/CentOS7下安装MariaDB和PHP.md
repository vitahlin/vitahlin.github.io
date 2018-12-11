---
title: Centos7下安装MariaDB和PHP
tags: Database
categories: Database
comments: true
copyright: true
abbrlink: install_mariadb_and_php_in_centos7 
date: 2017-03-02 13:33:33
updated: 2017-03-02 13:33:33
---

### MariaDB

#### 安装 

通过 `yum` 安装：

```c 
yum -y install mariadb mariadb-server
```

#### 启动

```c 
// 启动
systemctl start mariadb

// 查看运行状态
systemctl status mariadb

// 设置开机启动 
systemctl enable mariadb
```

<!--more-->

#### 设置

**进入设置前要先用 `systemctl start mariadb` 启动服务才行。**

运行如下命令进行设置：

```
// 拷贝配置文件（如果/etc目录下面默认有一个my.cnf，直接覆盖即可）
cp /usr/share/mysql/my-huge.cnf /etc/my.cnf

// 命令行中输入下述命令进行设置
mysql_secure_installation
```


设置过程：

```c 
// 输入root密码，初次登陆时直接Enter
Enter current password for root (enter for none):

// 是否设置root密码，我们选择不设置
Set root password? [Y/n]

// 删除匿名用户
Remove anonymous users? [Y/n] y 

// 是否允许root用户远程连接
Disallow root login remotely? [Y/n] y 

// 删除测试数据库
Remove test database and access to it? [Y/n] y 

// 重新加载授权信息
Reload privilege tables now? [Y/n] y 
```

完成后，即可登录。


### PHP

安装：

```c 
// 安装PHP
yum install php

// 安装PHP组件，使PHP支持 MariaDB
yum install php-mysql php-gd libjpeg* php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-bcmath php-mhash
```

### Apache 

安装：

```c 
yum install httpd // 安装
```

启动：

```c 
systemctl start httpd.service  // 启动apache
systemctl stop httpd.service  // 停止apache
systemctl restart httpd.service // 重启apache
systemctl enable httpd.service // 设置apache开机启动
```

### 参考链接
- [https://zhidao.baidu.com/question/327964276021131245.html](https://zhidao.baidu.com/question/327964276021131245.html)