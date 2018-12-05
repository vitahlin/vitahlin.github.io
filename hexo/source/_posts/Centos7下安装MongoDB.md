---
title: Centos7下安装MongoDB
tags: Database
categories: Database
comments: true
copyright: true
abbrlink: install-mongodb-in-centos7
date: 2017-03-02 20:27:33
updated: 2017-03-02 20:27:33
---

### 通过`yum`安装 

#### 配置`yum`


用 vim 创建文件：
```c
vi /etc/yum.repos.d/mongodb-org-3.4.repo
```

填入内容：
```c 
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
```

<!--more-->

#### 安装
通过以下命令安装： 
```c 
yum install -y mongodb-org
```

安装过程中出现验证错误的情况，则取消`gpgcheck`的验证。修改`mongodb-org-3.4.repo`文件后重新安装：
```c 
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=0
enabled=1
```

#### 相关文件位置

安装完后配置文件在：`/etc/mongod.conf`  

数据文件在：`/var/lib/mongo`  

日志文件在：`/var/log/mongodb`

#### 启动服务

可以使用相关命令来启动或者关闭 MongoDB:
```c 
// 启动 
systemctl start  mongod.service

// 检查是否启动
systemctl status  mongod.service

// 关闭 
systemctl stop  mongod.service

// 重启服务
systemctl restart  mongod.service

// 设置开机启动
systemctl enable  mongod.service
``` 

#### 访问权限控制 

MongoDB服务默认只绑定在本机IP上，即只有本机才能访问MongoDB，我们可以修改访问权限控制让外网也能访问。

修改配置文件`/etc/mongod.conf`将其中的`bindip:127.0.0.1`注释即可。

#### 参考链接
- MongoDB 访问控制 [http://blog.csdn.net/mniwc/article/details/8567197](http://blog.csdn.net/mniwc/article/details/8567197)
- Centos7 安装 MongoDB [http://www.centoscn.com/CentosServer/sql/Mariadb/2015/0503/5342.html](http://www.centoscn.com/CentosServer/sql/Mariadb/2015/0503/5342.html)
- MongoDB官网 [https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/)