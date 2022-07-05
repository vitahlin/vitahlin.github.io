---
title: Mac下利用alias简化命令
tags: Linux&Unix
categories: Linux&Unix
comments: true
abbrlink: use_alias_in_mac
date: 2016-12-06 21:07:33
updated: 2016-12-06 21:07:33
copyright: true
---

**系统版本macOS 10.12.1**

Alias是linux中常用的别名命令，当有一些比较复杂的命令需要经常执行的时候，alias对效率的提升立竿见影。

### 编辑`.bash_profile`或者`.zshrc`
如果Mac终端是使用`bash`的话就设置`.bash_profile`文件，如果是使用`zsh`的话就设置`.zshrc`文件。
这两个文件一般都放在`~`目录下，用命令`ls -a`即可查看隐藏文件，如果不存在这两个文件，用Vim新建即可。


### 设置命令别名
`alias`用法如下：
```c
alias [别名]='[指令名称]'
```

<!--more-->

**注意等号两边均无空格**，指令名称中如有空格，需用引号包裹，例如：
```c
alias ll='ls -l'
```

如，我们平常要登陆本地168服务器，可以设置`alias ssh168='ssh root@192.168.0.168'`，在终端中输入命令`ssh168`即等同于`ssh root@192.168.0.168`。

### 生效
在`.bash_profile`或者`.zshrc`文件中设置命令别名后，还需要在终端中执行如下命令使之生效：
```c 
source ~/.bash_profile
source ~/.zshrc 
```

要查看自定义的命令在终端中输入`alias`即可。
