---
title: Wireshark抓取本地TCP回环包
tags: Network
categories: Network
comments: true
copyright: true
abbrlink: tcp_sequence_number
date: 2018-12-15 01:47:33
updated: 2018-12-15 01:47:33
---

### 概述

`Wireshark`（前称`Ethereal`）是一个网络数据包分析软件。网络数据包分析软件的功能是截取网络数据包，并尽可能显示出最为详细的网络数据包数据。`Wireshark`使用`WinPCAP`作为接口，直接与网卡进行数据报文交换。

我们在开发TCP程序的时候，可以在本地通过`Wireshark`抓包，进行数据的验证。这里介绍`Wireshark`的一些基本用法。

### 安装`Wireshark`

通过`Homebrew`安装：
```c 
brew cask install wireshark
```

### 界面介绍

初始界面：
![]()