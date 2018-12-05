---
title: Centos7设置静态IP
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: set-static-ip-in-centos7
date: 2017-01-19 11:28:33
updated: 2017-01-19 11:28:33
---

### 修改配置文件

进入`/etc/sysconfig/network-scripts`目录，找到网络接口的配置文件（这里以ifcfg-enp0s3为例），打开该文件，修改配置：

```c
BOOTPROTO=static
IPADDR=192.168.0.169
NETMASK=255.255.255.0
NM_CONTROLLED=no
ONBOOT=yes
```

<!--more-->

`NM_CONTROLLED=no`表示该接口将通过该配置文件进行设置，而不是通过网络管理器进行管理。

`IPADDR`是要设置的静态IP地址。

`ONBOOT=yes`表示系统将在启动时开启该接口。

### 重启网络服务
使用命令重启网络服务：
```c
systemctl restart network.service
```

可以使用命令`ip add`查看设置的静态IP是否生效。

### 参考链接
- [http://www.linuxidc.com/Linux/2014-10/107789.htm](http://www.linuxidc.com/Linux/2014-10/107789.htm)