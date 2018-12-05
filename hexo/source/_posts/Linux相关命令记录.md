---
title: Linux相关命令记录
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: some_command_in_linux
date: 2017-01-02 20:02:33
updated: 2017-01-05 20:18:33
---

### 查看系统信息

1. 查看系统版本
```c 
more /etc/redhat-release

// 显示结果
CentOS Linux release 7.0.1406 (Core)
```

2. 查看正在运行的内核版本
```c
cat /proc/version

// 显示结果
Linux version 3.10.0-123.9.3.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.2 20140120 (Red Hat 4.8.2-16) (GCC) ) #1 SMP Thu Nov 6 15:06:03 UTC 2014
```

3. 显示电脑以及操作系统的相关信息
```c
uname -a

// 显示结果
Linux iZ23dnc7nkxZ 3.10.0-123.9.3.el7.x86_64 #1 SMP Thu Nov 6 15:06:03 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

<!--more-->

### `mv` 重命名或者移动

#### 重命名文件或目录
```c 
mv xxx_1 xxx_2
```
`xxx_1`和`xxx_2`既可以是文件也可以是目录

#### 移动文件
把`file_1.txt`文件从当前目录移动到目录，以`/home/test/`为例：
```c 
mv file_1.txt /home/test
```

带参数的命令示例：
```c 
mv -i file_1.txt /home/test
mv -uv *.txt /home/test
```

- `-i` 当存在同名文件时，会提示我们关于是否覆盖
- `-n` 将不会允许我们覆盖任何已存在的文件
- `-u` 只在源文件比目标文件新时才执行更新
- `-b` 该选项会在新文件覆盖旧文件时将旧文件做备份
- `-f` 不进行任何提示而覆盖
- `-v` 打印移动信息，当移动或重命名一大堆文件或目录时，不想去目标位置去查看的情况下知道命令是否成功地执行了就需要用到该参数。

#### 移动多个文件
同移动一个文件类似，多个文件名之间用空格隔开：
```c 
mv file_2.txt file_3.txt file_4.txt /home/test
```

#### 移动目录
同移动文件类似，把文件名换成目录名称即可：
```c 
mv temp/ /home/test/
```
即把`temp`目录移动到`/home/test/`目录下。

### `rm` 删除文件

使用rm命令要格外小心。因为一旦删除了一个文件，就无法再恢复它。

语法：
```c 
rm -i test
```

`test`可以是文件，也可以是目录。删除多个文件的时候，文件之间以空格分开即可。

参数说明
- `-i` 进行任何删除操作前必须先确认
- `-f` 强制删除文件或目录
- `-r/R` 同时删除该目录下的所有目录层
- `-v ` 显示指令的详细执行过程
- `-d` 直接把欲删除的目录的硬连接数据删除成0，删除该目录

### 查看IP地址

CentOS7 查看 IP 地址：
`ip addr`

### 修改密码 

以 root 登录，输入 `passwd`进行修改密码。
