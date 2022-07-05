---
title: Vim插件The-NERD-tree安装和配置
tags:
  - Linux&Unix
  - Vim
categories: Vim
comments: true
copyright: true
abbrlink: 3079a454
date: 2017-07-17 16:21:33
updated: 2016-07-17 16:21:33
---

### 安装
用插件`Vundle`安装：
```c
Plugin 'The-NERD-tree'
```

### 设置
在`.vimrc`文件中增加配置：

```c
" 设置开启和关闭快捷键
nmap <silent> <F2> :NERDTreeMirror<CR>
nmap <silent> <F2> :NERDTreeToggle<CR>

"窗口大小
let NERDTreeWinSize=25 

"窗口位置
let NERDTreeWinPos='left'

"是否默认显示行号
let NERDTreeShowLineNumbers=1

"是否默认显示隐藏文件
let NERDTreeShowHidden=0
```