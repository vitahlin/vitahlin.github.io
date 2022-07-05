---
title: macOS上安装Git和SourceTree
tags: Git
categories: Git
comments: true
copyright: true
abbrlink: install_git_and_sourcetree_in_macos
date: 2017-01-04 20:45:33
updated: 2017-01-04 20:45:33
---

**系统版本：macOS 10.12.2**

SourceTree是一个免费的git客户端管理工具。macOS系统上默认会安装Git，我们可以自己安装Git和SourceTree。
安装前，可以查看系统默认git版本:
```c
git --version
git version 2.9.3 (Apple Git-75)
```

<!--more-->

### 使用brew安装git和SourceTree

安装命令：
```c
brew install git
brew cask install soucetree
```

安装完后，我们运行命令`git --version`仍然会显示系统默认版本的Git，我们需要修改 `.bash_profile` 文件，在 `.bash_profile` 文件中新增以下内容：
```c
export PATH=/usr/local/bin:/usr/local/sbin:${PATH}
```

上述命令可以让bash优先搜索`/usr/local`下的指令，而且不会覆盖老文件。然后用命令`source .bash_profile`更新文件，接下来我们在终端中查看git版本就会显示我们安装的那个版本。

用`brew info git`查看安装的git信息时提示gettext未安装，可以用如下命令重装git:
```c
brew install git --with-gettext
```

### SourceTree可能出现问题

#### 提示`warning: templates not found /usr/local/git/share/git-core/templates`

在使用SourceTree时有可能会出现`warning: templates not found /usr/local/git/share/git-core/templates `的问题，解决方案：
运行如下命令：
```c
sudo mkdir -p /usr/local/git/share/git-core/templates
sudo chmod -R 755 /usr/local/git/share/git-core/templates
```

参考链接
- [https://segmentfault.com/q/1010000000095119](https://segmentfault.com/q/1010000000095119)
- [http://www.jianshu.com/p/cdf39f056a86](http://www.jianshu.com/p/cdf39f056a86)

#### 提示`RPC failed; curl 56 SSLRead() return error`错误

在克隆项目时有时候会出现`RPC failed; curl 56 SSLRead() return error`错误，错误详情：
```c
Cloning into '/Users/vitah/Downloads/Code/SeaBattle/SeaBattleClient'...
remote: Counting objects: 45608, done.
remote: Compressing objects: 100% (17233/17233), done.
error: RPC failed; curl 56 SSLRead() return error -36.00 KiB/s
fatal: The remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```

解决方案：
运行`git config --global http.postBuffer 524288000`后再进行Clone。


参考链接
- [http://stackoverflow.com/questions/6842687/the-remote-end-hung-up-unexpectedly-while-git-cloning](http://stackoverflow.com/questions/6842687/the-remote-end-hung-up-unexpectedly-while-git-cloning)

#### 提示`RPC failed; curl 18 transfer closed……`错误

错误详情：
```c 
Cloning into '/Users/vitah/Downloads/Code/SeaBattle/SeaBattleDoc'...
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
fatal: The remote end hung up unexpectedly
```

解决，重装git:
```c
brew reinstall git --with-brewed-curl --with-brewed-openssl
```

参考链接
- [http://stackoverflow.com/questions/30385939/git-clone-fails-with-sslread-error-on-os-x-yosemite](http://stackoverflow.com/questions/30385939/git-clone-fails-with-sslread-error-on-os-x-yosemite)
