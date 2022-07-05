---
title: golint使用
tags: Go 
categories: Go
comments: true
copyright: true
abbrlink: use_golint
date: 2018-12-12 10:05:33
updated: 2018-12-12 10:05:33
---

##### 安装
百度很多命令是错误的，正确的安装命令如下：
```c
go get -u github.com/golang/lint/golint
```

##### 使用

检测文件或者目录：
```c
golint xxx.go 
golint dir
```