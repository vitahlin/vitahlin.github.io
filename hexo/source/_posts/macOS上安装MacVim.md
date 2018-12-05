---
title: macOS上安装MacVim
tags: [Linux&Unix,Vim]
categories: Vim
comments: true
abbrlink: install_macvim_in_macos
date: 2016-11-28 22:28:33
updated: 2016-11-28 22:28:33
copyright: true
---

**系统版本macOS 10.12.1，默认安装`Homebrew`**

### 安装

通过`brew`安装，执行命令`brew install macvim`进行安装。
`brew`的用法可以查看介绍：[[链接地址]](http://www.vitah.net/posts/28edc106/)

<!--more-->

### `.vimrc`设置

#### `.vimrc`文件位置
打开MacVim-Edit-Startup Settings，可以打开~/.vimrc，添加配置。在Vim中输入:version可以查看系统中vimrc的位置。会显示如下信息，分别是系统配置文件和用户配置文件：
```c 
system vimrc file: "$VIM/vimrc"
user vimrc file: "$HOME/.vimrc"
```

如果不知道$VIM或$HOME具体是哪个目录,可以在Vim中用下面的命令查看:`:echo $VIM`和`:echo $HOME`。

#### 基本的设置

可以在`.vimrc`文件中增加一些基本的设置：
```c 

" 设置代码折叠根据语义折叠
set foldmethod=syntax
" 设置vim开启时不开启折叠
set nofoldenable

" 共享剪贴板
set clipboard+=unnamed

" 显示行号
set number 

" 设置字体及其大小
set guifont=Source_Code_Pro:h15

" 字符编码相关设置
set termencoding=utf-8
set encoding=utf-8
set fileencodings=utf8,ucs-bom,gbk,cp936,gb2312,gb18030

" 设置命令行高度
set cmdheight=2

" 在编辑过程中，在右下角显示光标位置的状态行
set ruler

" 显示命令
set showcmd

" autowrite
set autowrite
set autowriteall

" autoread
set autoread


set confirm

" 光标移动到buffer的顶部和底部时保持3行距离
set scrolloff=3

set wildmenu
set history=50

" 在Visual模式时，按Ctrl+c复制选择的内容
vmap <C-c> "+y

" 字符间插入的像素行数目
set linespace=0

" 键入闭括号时显示它与前面的那个开括号匹配
set showmatch
set matchtime=2

" 语法高亮
syntax on
syntax enable

" 用浅色高亮当前行
autocmd InsertLeave * se nocul
autocmd InsertEnter * se cul

" search
set hlsearch
set incsearch

" 智能对齐
set smartindent
set autoindent

" bakcspace
set backspace=eol,start,indent

" backup 不进行备份
set nobackup
set nowb
set noswapfile

" tab
set tabstop=4
set softtabstop=4
set shiftwidth=4
set noexpandtab

" 主题设置
syntax enable
set background=dark
colorscheme solarized
```

### 插件安装

在这边只介绍管理插件的插件`Vundle`和状态栏美化插件`vim-powerline`，这两个我觉得是必须的，而其他的插件，在我看来有些不一定需要，保持简单，在需要时再去安装即可，而不是在开始去安装一大堆插件，毕竟Vim的核心不在于插件。
如果你想要更多的插件安装以及设置可以查看我在Github上的介绍：**[Vim](https://github.com/vitahlin/Vim)**


#### 管理插件的插件[`Vundle`](https://github.com/vitahlin/Vim/tree/master/Vundle)

`Vundle`可以用于管理插件，方便的进行安装、更新和卸载。

##### 安装
在终端执行命令
```c 
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

这样的话，就设置Vundle的插件安装目录是`~/.vim/bundle/`。如果有需要可以自行修改，不过要注意的是修改此目录的话，`vimrc`文件中关于`Vundle`的那个路径设置也要做对应的更改。

`git clone`完成在`.vimrc`文件中添加必要的设置：
```c 
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" 在这里填写需要安装的插件

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
```

重启Vim后就可以在Vim中方便的用命令安装和卸载插件
```c 
PluginInstall // 安装插件
PluginClean   // 卸载插件
PluginList    // 显示已经安装的插件
```

#### 让Vim支持多文件编辑[`minibufexpl`](https://github.com/vitahlin/Vim/tree/master/minibufexpl)

##### 安装
使用Vundle进行安装：`Plugin 'fholgado/minibufexpl.vim'`

##### 设置
在.vimrc文件中增加配置：
```c 
" minibufexpl 多文件编辑
"------------------------------------------------------
"按下Ctrl+h/j/k/l，可以切换到当前窗口的上下左右窗口
"let g:miniBufExplMapWindowNavVim = 1   

"按下Ctrl+箭头，可以切换到当前窗口的上下左右窗口
"let g:miniBufExplMapWindowNavArrows = 1 

"启用以下两个功能：Ctrl+tab移到下一个窗口
"let g:miniBufExplMapCTabSwitchBufs = 1 

"不要在不可编辑内容的窗口（如TagList窗口）中打开选中的buffe
let g:miniBufExplModSelTarget=1  
"-----------------------------------------------------
```

#### 状态栏美化[`vim-powerline`](https://github.com/vitahlin/Vim/tree/master/vim-powerline)

##### 安装
用Vundle安装：
```c 
Plugin 'Lokaltog/vim-powerline'
```

##### 设置
在.vimrc文件中增加配置：
```c 
" vim-powerline 状态栏美化 
"---------------------------------------
let g:Powerline_symbols = 'fancy'
set laststatus=2
"---------------------------------------
```

##### 可能出现的问题
当出现乱码或者状态栏不正确显示箭头时需要安装对应的字体进行解决。可以参考链接：[链接地址](https://github.com/vitahlin/Vim/tree/master/vim-powerline)
