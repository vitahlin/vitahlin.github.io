---
title: macOS下的tree命令
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: command_tree
date: 2017-02-09 10:25:33
updated: 2017-02-09 10:25:33
---

**系统版本macOS 10.12.3**

Linux下的`tree`命令可以很方便的以树状图列出文件目录结构(注：某些Linux上可能没有tree命令)，但是在macOS下，我们则需要安装工具。

### 安装tree

在macOS中通过`Homebrew`安装`tree`： 
```c 
brew install tree 
```

安装成功后，我们直接在目录中执行`tree`即可查看该目录的结构树。

<!--more-->

### tree命令相关参数

#### 指定目录层级
当目录层级太多时，可以用`-L`指定目录层级：
```c 
tree -L 2
```

#### 只显示目录

```c 
tree -d
```

### 参考链接

- [http://yijiebuyi.com/blog/c0defa3a47d16e675d58195adc35514b.html](http://yijiebuyi.com/blog/c0defa3a47d16e675d58195adc35514b.html)
- Linux tree命令介绍 [http://www.runoob.com/linux/linux-comm-tree.html](http://www.runoob.com/linux/linux-comm-tree.html)
