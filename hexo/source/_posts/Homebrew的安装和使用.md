---
title: Homebrew的安装和使用
tags: Linux&Unix
categories: Linux&Unix
comments: true
abbrlink: install_homebrew
date: 2016-11-26 18:12:33
updated: 2016-11-26 18:12:33
copyright: true
---

**系统版本macOS 10.12.1**

### `Homebrew`是什么

`Homebrew`官网：[http://brew.sh/index_zh-cn.html](http://brew.sh/index_zh-cn.html)
“Homebrew installs the stuff you need that Apple didn’t.——Homebrew 使 OS X 更完整”。
官网简单明了地介绍了如何安装和使用这个工具，并提供了自己的Wiki。

### Mac安装`Homebrew`

`Homebrew`的安装很简单，使用一条Ruby命令即可，在终端输入：
```c
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
就可以安装。安装`Homebrew`的详细说明可以参考其官网上的介绍。

<!--more-->

### 使用`Homebrew`安装软件

安装完后可以用`brew install xxx`安装需要软件，也可以用`brew cask install xxx`安装带有GUI的某些软件，比如网易云音乐，谷歌浏览器等。 

一些常用命令：
```c 
brew search xxx // 搜索
brew cask search xxx // 搜索
brew update // 更新brew
brew upgrade // 更新用brew安装的软件
brew cleanup // 清除安装包
brew cask cleanup // 清除安装包
brew doctor // 检测
brew outdated // 看一下哪些软件可以升级
brew info xxx // 查看某个软件信息
brew cask info xxx // 查看某个软件信息
```

### `brew`和`brew cask`的差别

可以参看这个链接：[https://www.zhihu.com/question/22624898](https://www.zhihu.com/question/22624898)

简单的说，`brew`主要用来下载一些不带界面的命令行下的工具和第三方库来进行二次开发。`brew cask`主要用来下载一些带界面的应用软件，下载好后会自动安装，并能在mac中直接运行使用.`brew cask`可以看作是对App Store的补充。安装带有界面的软件时我们可以优先在 App Store上搜索，没有的话再去`brew cask`里搜索。
值得说明的是，现在`brew cask`暂时没有更新功能，需要更新到话我们可以对该软件进行重装。

### 可能出现的问题

#### `brew update`卡住并且出现错误
使用命令`brew update`更新时，可能出现如下错误：
```c
fatal: unable to access 'https://github.com/Homebrew/brew/': SSLRead() return error -9806
fatal: unable to access 'https://github.com/Homebrew/homebrew-core/': SSLRead() return error -9806
fatal: unable to access 'https://github.com/caskroom/homebrew-cask/': SSLRead() return error -9806
Error: Fetching /usr/local/Homebrew failed!
Fetching /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core failed!
Fetching /usr/local/Homebrew/Library/Taps/caskroom/homebrew-cask failed!
```

这是因为国内的网络原因，可以改为如下命令：`brew update --verbose`。或者切换良好的网络环境，良好的，你懂的。

##### 参考链接
- [https://github.com/Homebrew/legacy-homebrew/issues/46590](https://github.com/Homebrew/legacy-homebrew/issues/46590)

#### `brew install`安装失败

同样是因为国内特殊的网路环境，可以切换良好的网络环境或者安装**proxychains-ng**。`proxychains-ng`用法的介绍可参考这里：[[链接地址](http://www.vitah.net/posts/745a6d7/)]