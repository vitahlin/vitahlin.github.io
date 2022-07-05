---
title: macOS上Node.js环境的安装
tags: Linux&Unix
categories: Node.js
comments: true
abbrlink: install_nodejs_in_macos
date: 2016-11-29 17:49:33
updated: 2016-11-29 17:49:33
copyright: true
---

**系统版本macOS 10.12.1，默认安装`Homebrew`**

### 安装nvm

通过`brew`安装，执行命令`brew install nvm`进行安装。
nvm是用来管理Node版本的工具。

安装完后还需要进行设置，创建nvm的工作目录，用来保存下载的各个node版本。
```c 
// 创建nvm的工作目录
mkdir ~/.nvm
```

创建完后要在`.zshrc`或者`.bash_profile`文件中加入环境变量的设置，加入以下内容：
```c 
export NVM_DIR="$HOME/.nvm"
. "/usr/local/opt/nvm/nvm.sh"
```
完成上述步骤后可以用`nvm --version`查看nvm版本。

<!--more-->

### 安装Node

用命令`nvm ls-remote`查看可安装的Node版本，有时候因为网络问题可能只会显示iojs-x.x.x的版本，多尝试几次命令即可。

查看列表后，可以安装想要安装的版本，例如：`nvm install v5.12.0`，安装完成后，可以用命令`node -v`查看Node.js版本。

### 相关命令
```c 
npm root  // 查看当前包的安装路径
npm root -g // 查看全局包的安装路径
npm uninstall xxx // 卸载模块
```
