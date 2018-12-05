---
title: Centos7设置开机自动联网
tags: Linux&Unix
categories: Linux&Unix
comments: true
abbrlink: set-automatically-connect-network-in-centos7
date: 2016-11-29 17:04:33
updated: 2016-11-2in-centos7
copyright: true
---

Centos7系统开机是默认没有联网的，可以修改网卡的配置文件，实现Centos7的开机自动联网。

### 打开配置文件
```c
cd /etc/sysconfig/network-scripts/
vi ifcfg-enp2s0
```

<!--more-->

###  修改配置文件内容
```c
TYPE=Ethernet
Last login: Mon Sep 26 10:01:09 on ttys000
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=enp2s0
UUID=293bb84c-d00f-4b86-9e05-4bf8b768c76a
DEVICE=enp2s0
ONBOOT=no
DNS1=8.8.8.8
PEERDNS=yes
PEERROUTES=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
```

**将原先的`ONBOOT=no`改为`ONBOOT=yes`，然后保存重启电脑即可。**