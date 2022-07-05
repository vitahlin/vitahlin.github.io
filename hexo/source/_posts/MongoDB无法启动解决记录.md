---
title: MongoDB无法启动解决记录
tags: Database
categories: Database
comments: true
copyright: true
abbrlink: resolve_issus_that_mongodb_can_not_open
date: 2017-02-09 17:18:33
updated: 2017-02-09 17:18:33
---

**系统版本：Centos 6.5**


### 问题描述

某次，在重启服务器之后就发现mongodb数据库无法启动。
使用命令:
```c 
service mongod start
service mongod restart 
```
进行启动操作时，不显示`OK`也不显示`Failed`字样；进行重启操作时，只显示关闭正常，但是启动状态一样不明，不显示`OK`也不显示`Failed`。 

### 解决

由于不知道什么原因，我们可以去查看mongodb的`log`文件，查看打印内容。

<!--more-->

#### 查找log文件位置

我们从mongodb配置文件中找到`log`文件的目录位置：

```c 
vi /etc/mongod.conf
```

找到文件里面的systemlog配置：
```c 
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
```

可以看到，`log`文件的目录是`/var/log/mongodb/mongod.log`。 

#### 打开log文件，查看问题描述

打开文件，跳到文件末尾，可以看到输出内容：
```c 
2017-02-09T16:16:30.747+0800 I CONTROL  [main] ***** SERVER RESTARTED *****
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] MongoDB starting : pid=7202 port=27017 dbpath=/var/lib/mongo 64-bit host=serv2.skyfun.com
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] db version v3.2.4
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] git version: e2ee9ffcf9f5a94fad76802e28cc978718bb7a30
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] allocator: tcmalloc
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] modules: enterprise
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] build environment:
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten]     distmod: rhel62
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten]     distarch: x86_64
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten]     target_arch: x86_64
2017-02-09T16:16:30.753+0800 I CONTROL  [initandlisten] options: { config: "/etc/mongod.conf", net: { bindIp: "0.0.0.0", port: 27017 }, processManagement: { fork: true, pidFilePath: "/var/run/mongodb/mongod.pid" }, storage: { dbPath: "/var/lib/mongo", journal: { enabled: true } }, systemLog: { destination: "file", logAppend: true, path: "/var/log/mongodb/mongod.log" } }
2017-02-09T16:16:30.768+0800 E NETWORK  [initandlisten] listen(): bind() failed errno:98 Address already in use for socket: 0.0.0.0:27017
2017-02-09T16:16:30.768+0800 E NETWORK  [initandlisten]   addr already in use
2017-02-09T16:16:30.768+0800 E STORAGE  [initandlisten] Failed to set up sockets during startup.
2017-02-09T16:16:30.768+0800 I CONTROL  [initandlisten] dbexit:  rc: 48
2017-02-09T16:24:00.561+0800 I CONTROL  [main] ***** SERVER RESTARTED *****
```

打印出的 **SERVER RESTARTED **说明这是我们进行重启操作时输出的`log`内容。

#### 定位问题

可以看到 **listen(): bind() failed errno:98 Address already in use for socket: 0.0.0.0:27017** 这行内容，提示已经在使用，我们可以去验证这个端口是否已经被占用。

查找被占用的端口：
```c 
netstat -tln
netstat -tln | grep 27017
```

第一行命令可以查看端口使用情况，第二行命令则只查找 **27017** 端口的使用情况。

如果端口被占用，接下来我们可以用`lsof`查看端口属于那个程序，被哪个进程占用。(注：Linux中若没有安装lsof，则可以安装一下，centos安装命令: yum install lsof)

执行`lsof -i:27017`后，我们发现，这个端口就是被mongodb本身占用，所以我们需要释放这个端口，根据查找出来的信息，杀掉占用端口的进程：
```c 
kill -9 1639 // 1639是进程的pid 
```

完成后，我们再次重新启动数据库，显示重启`OK`。

重启OK之后，客户端进行连接时发现，仍然失败。尝试用Robomongo进行连接数据库时也提示连接失败，可以得知应该是防火墙问题，采用命令关掉防火墙即可：
```c 
service iptables stop
```

注：这里因为是开发服务器，所以直接关闭防火墙没有大碍，正常情况下选择开启某些端口而不是关闭整个防火墙更为妥当。


### 参考链接
- 查看端口占用 [http://linux.it.net.cn/e/Linuxit/system/2014/1104/7822.html](http://linux.it.net.cn/e/Linuxit/system/2014/1104/7822.html)
- Mongodb错误解决 [http://stackoverflow.com/questions/37483014/mongodb-failed-to-set-up-sockets-during-startup](http://stackoverflow.com/questions/37483014/mongodb-failed-to-set-up-sockets-during-startup)
