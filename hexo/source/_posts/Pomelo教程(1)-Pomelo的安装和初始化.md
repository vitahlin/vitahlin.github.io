---
title: Pomelo教程(1)-Pomelo的安装和初始化
tags:
  - Pomelo
  - Node.js
categories: Pomelo
comments: true
copyright: true
abbrlink: pomelo_series_install_pomelo_and_init
date: 2016-12-06 22:41:33
updated: 2016-12-06 22:41:33
---

**系统版本macOS 10.12.1，文章内容均在该环境下测试通过。**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

### Pomelo介绍
Pomelo是网易开发用Node.js写的一个游戏服务器框架，使用简单，它包括基础开发框架和一系列相关工具和库，可以帮助开发者省去游戏开发中枯燥的重复劳动和底层逻辑工作，免除开发者的重造轮子，让开发者可以更多地去关注游戏的具体逻辑，大大提高开发效率。更多的介绍可以查看官网链接。

官网：[http://pomelo.netease.com/](http://pomelo.netease.com/)

### Pomelo安装

因为**Pomelo是基于Node.js**的，所以要确保系统上已经安装了Node，另外，因为pomelo是用Javascript写成，但是pomelo依赖的库中，有使用了C++语言写的扩展，因此安装pomelo的过程中会使用到C++编译器。所以在macOS系统上,则需要安装`Xcode Command Line Tools`或者Xcode的完整包以及make工具.

<!--more-->

#### 安装nvm和node

macOS上安装nvm和node可以参考这个链接：[http://www.vitah.net/posts/51475956/](http://www.vitah.net/posts/51475956/)

安装完后可以查看nvm和node版本：
```c 
vitah@VitahdeMacBook-Pro  ~  nvm --version
0.32.1
vitah@VitahdeMacBook-Pro  ~  node --version
v5.12.0
```
如上所示，这边用于开发的`nvm`版本是`0.32.1`，`node`版本是`v5.12.0`。

#### 安装Pomelo

安装完nvm和node后，我们需要安装`pomelo`模块，关于node模块全局安装和本地安装的区别可以参照：[链接](https://segmentfault.com/q/1010000002467962)

为了能在终端中使用`pomelo`的相关命令，我们需要用如下命令**全局安装**`pomelo`模块：
```javascript
npm install pomelo -g
```

安装完后，可以查看`pomelo`版本：
```javascript
vitah@VitahdeMacBook-Pro  ~  pomelo --version
1.2.3
```

这里用于开发的`Pomelo`版本是`1.2.3`。

#### 初始化项目

安装完`pomelo`模块，我们需要用`pomelo`初始化一个项目：
```c
mkdir VitahPomeloserver
cd VitahPomeloserver
pomelo init
```
创建一个目录`VitahPomeloserver`，然后用`pomelo init`进行初始化，或者直接用命令`pomelo init ./VitahPomeloserver`初始化。

在执行`pomelo init`命令后，它会提示如下内容：
```c 
The default admin user is:
  username: admin
  password: admin
You can configure admin users by editing adminUser.json later.
Please select underly connector, 1 for websocket(native socket), 2 for socket.io, 3 for wss, 4 for socket.io(wss), 5 for udp, 6 for mqtt: [1]
```
让你选择要选用的网络连接方式，这里我们选择`2 for socket.io`，输入`2`即可。

#### 安装Pomelo所需模块

初始化后，`pomelo`会在目录下生成多个文件，我们需要运行脚本`npm-install.sh`来安装需要模块，因为国内环境时间可能要久一点，等待安装完即可。

到这里，我们就完成对一个项目的初始化了，接下来就可以在`pomelo`框架的基础上进行游戏服务端的开发了。

项目在Github上的地址：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)

### 参考链接
- [https://github.com/NetEase/pomelo/wiki/%E5%AE%89%E8%A3%85pomelo](https://github.com/NetEase/pomelo/wiki/%E5%AE%89%E8%A3%85pomelo)
- [https://segmentfault.com/q/1010000002467962](https://segmentfault.com/q/1010000002467962)
