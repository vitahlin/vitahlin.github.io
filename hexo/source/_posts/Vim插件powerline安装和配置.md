---
title: Vim插件powerline安装和配置
tags:
  - Linux&Unix
  - Vim
categories: Vim
comments: true
copyright: true
abbrlink: da7fe432
date: 2017-07-18 22:07:33
updated: 2016-07-18 22:07:33
---

### GitHub链接

[https://github.com/Lokaltog/vim-powerline](https://github.com/Lokaltog/vim-powerline)

### 安装

用插件`Vundle`安装：
```c
Plugin 'Lokaltog/vim-powerline'
```

### 设置
在`.vimrc`文件中增加配置：

```c
let g:Powerline_symbols = 'fancy'
set laststatus=2
```

<!--more-->

### 可能出现的问题

#### 出现乱码

因为缺少字体，需要安装Powerline的相关字体：[https://github.com/powerline/fonts](https://github.com/powerline/fonts)

#### 无法正确显示箭头

这因为是缺少对应字体，需要安装如下字体： [https://github.com/vitahlin/consolas-powerline-vim](https://github.com/vitahlin/consolas-powerline-vim) 在字体库中安装时，可能会出现警告提示，忽略，直接安装后重启Vim即可正确显示。