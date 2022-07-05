---
title: Vim插件YouCompleteMe介绍
tags:
  - Linux&Unix
  - Vim
categories: Vim
comments: true
copyright: true
abbrlink: 15cf7230
date: 2017-07-22 14:37:33
updated: 2016-07-22 14:37:33
---

### 介绍
Github:[https://github.com/Valloric/YouCompleteMe](https://github.com/Valloric/YouCompleteMe)

用于代码补全和提示，非常强大，支持C、C++、JavaScript、Python、Go等等。该插件需要Python支持。具体可参考Github上说明。

### 安装

#### 用插件`Vundle`安装

```c
Plugin 'Valloric/YouCompleteMe'
```
因为这个插件很大，所以安装过程会持续很久。
可以到目录下`.vim/bundle/YouCompleteMe`，然后执行命令`du -s`来查看文件夹大小，会看到大小在缓慢增加，就表示文件在下载过程中。

<!--more-->

#### 用`git clone`安装

觉得用`Vundle`没有进度条不够直观，可以用`Git`直接下载，执行如下命令(目录随自己目录更改):
```c
git clone https://github.com/Valloric/YouCompleteMe.git ~/.vim/bundle/YouCompleteMe
cd ~/.vim/bundle/YouCompleteMe  
git submodule update --init --recursive 
```

注意，`git submodule update --init --recursive`时必须要执行的才能保证下载成功。

### 编译构建 ycm_core 库

进入到YCM插件目录，执行`install.py`脚本：
```c
cd ~/.vim/bundle/YouCompleteMe
./install.py
```

为了添加对不同语言的支持，对在编译时添加不同的参数。

值得说明的是，如果你开始用于C/C++开发，执行了
```c
./install.py --clang-completer
```

后续也想用于JavaScript开发，只需执行如下命令：
```c
./install.py --tern-completer
```
按上述命令安装`tern`即可，而不需要再次指定对C/C++的支持参数：
```c
// 没必要
./install.py --clang-completer --tern-completer
```

#### 添加对C/C++的支持

需要执行如下命令：
```c
./install.py --clang-completer
```

并且需要下载 `Command Line Tools`，可以使用如下命令下载：
```c
xcode-select --install
```

#### 添加对JavaScript的支持
`install`命令如下：
```c 
./install.py --tern-completer
```

### 配置

#### `vimrc`配置参考
```c
" 开启语义补全
let g:ycm_seed_identifiers_with_syntax=1

"在注释输入中也能补全
let g:ycm_complete_in_comments=1
let g:ycm_collect_identifiers_from_tags_files=1
let g:ycm_min_num_of_chars_for_completion=1

"在字符串输入中也能补全
let g:ycm_complete_in_strings = 1

let g:ycm_filetype_blacklist = {
      \ 'tagbar' : 1,
      \ 'nerdtree' : 1,
      \}

" 设置默认的.ycm_extra_conf.py文件
let g:ycm_global_ycm_extra_conf = '~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp/ycm/.ycm_extra_conf.py'
let g:ycm_enable_diagnostic_signs = 0
let g:ycm_enable_diagnostic_highlighting = 0
let g:ycm_confirm_extra_conf = 0
let g:ycm_cache_omnifunc=0
let g:ycm_server_keep_logfiles = 1

" 不弹出Scratch窗
set completeopt-=previe
```

#### C 的设置

不仅需要在编译时添加对 C 的支持，还需要在 C 项目的根目录添加`.ycm_extra_conf.py` 文件，可以在 C 项目根目录下运行如下命令直接下载我修改过的支持 C 项目的`.ycm_extra_conf.py`文件：
```c
wget --no-check-certificate https://raw.githubusercontent.com/vitahlin/Vim/master/YouCompleteMe/c/.ycm_extra_conf.py
```

#### C++ 配置

编译和安装Command Line Tools和对C项目的配置一样，但是 `.ycm_extra_conf.py` 文件有所不同。

需要在 C++ 项目的根目录添加 `.ycm_extra_conf.py` 文件，可以在 C++ 项目根目录下运行如下命令直接下载我修改过的支持 C++ 项目的 `.ycm_extra_conf.py` 文件：
```c
wget --no-check-certificate https://raw.githubusercontent.com/vitahlin/Vim/master/YouCompleteMe/cpp/.ycm_extra_conf.py
```

**注：`.ycm_extra_conf.py`文件不要格式**

#### Node.js 设置

还需要在项目的跟目录添加 .tern-project 文件，可以在 Node.js 项目根目录运行如下命令直接下载我提供的 .tern-project 文件：
```c
wget --no-check-certificate https://raw.githubusercontent.com/vitahlin/Vim/master/YouCompleteMe/js/.tern-project
```

此外还需要 tags 文件，使用如下命令用 ctags 生成 tags 文件，ctags可以用 brew 安装：
```c
ctags -R app/ --javascript-kinds=+f+m+p+v --fields=+l --extra=+q
```

Node.js 项目中目录很多，比如说模块目录 node_modules 就不需要生成 tags 文件，所以这里设置 `app/` 为指定要生成 tags 的目录，不设置指定目录的话，则默认对全部目录生成。 设置.vimrc文件，增加对tags文件的读取:
```c
let g:ycm_collect_identifiers_from_tags_files=1 
```

### 可能出现的问题

#### `ERROR: please install CMake and retry.`
运行`./install.py --clang-completer --tern-completer`时提示`"ERROR: please install CMake and retry."`。

则需要安装 CMake，可以通过 brew 安装：
```c
brew install cmake
```

#### 出现Read time out提示
在进行补全提示时，有时候 Vim 会出现 `Read time out`，具体内容：
```c
HTTPConnectionPool(host='localhost', port=37075): Read timed out. (read timeout=0.5)
```

可以修改 YouCompleteMe 中的 `TIMEOUT_SECONDS` 变量：
```c
cd ~/.vim/bundle/YouCompleteMe/python/ycm/client
vi completion_request.py
```
`completion_request.py` 文件**第34行** `TIMEOUT_SECONDS` 原来的值为0.5,改为2，如果改为2仍然出现问题可以改成30。改完后需要重启Vim。

#### 编译时下载Clang失败

用命令 `./install.py --clang-completer` 在 macOS 上编译时，当因为网络原因，Downloading Clang 4.0.1 时进度过慢，或者失败，**不能通过proxychains4来帮助下载，因为proxychains4可能会导致下载到错误的 Clang版本**，可以通过如下办法： 到 Clang 官网选择下载 Pre-Built Binaries Clang for Mac OS X，把下载的文件拷贝到YCM插件目录  `YouCompleteMe/third_party/ycmd/clang_archives` 下面，然后执行如下命令，重新编译：
```c
./install.py --clang-completer --system-libclang
```




