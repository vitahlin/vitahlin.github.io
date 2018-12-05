---
title: Pomelo教程(3)-客户端开发环境的介绍
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: 5af33cc8
date: 2017-02-14 11:04:33
updated: 2017-02-14 11:04:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

为了更直观的展现Pomelo框架的用法，我们可以搭建一个客户端用于测试和开发，以便更好的了解，这里我们以Unity3D来创建测试项目。

### Unity3D

Unity3D是现在广泛用于3D开发的一个专业引擎。关于它的更多信息可以查看它的官网介绍：[官网链接](https://unity3d.com/cn/)

关于它的安装和破解网上有很多教程，这里不再赘述。这里的开发版本是 **5.4.4f1**。


> 注：在macOS上进行Unity开发建议用 **5.4.4f1以上的版本** 。因为低版本的在运行时有时候会出现场景中所有物体都不显示的情况，只能重启Unity才能解决。

<!--more-->

### 代码编辑器

在macOS上进行Unity开发可以用Unity本身自带的编辑器：MonoDevelop，或者对Unity和VS Code进行配置以辅助开发(可以参考链接进行配置：[http://www.vitah.net/posts/82a25d54/](http://www.vitah.net/posts/82a25d54/))，这里以MonoDevelop为主。


### Unity3D所需插件

安装完Unity后，我们还需要几个插件来帮助我们进行开发。

#### pomelo-unityclient-socket

网易开发的用于Unity3D的客户端socket框架。在这里，我们用它实现和服务端的通信。关于它的更多介绍：[Github链接](https://github.com/NetEase/pomelo-unityclient-socket)

#### SimpleJson

Facebook开发的一款开源的json解析库，也是`pomelo-unityclient-socket`所依赖的。[Github上项目链接](https://github.com/facebook-csharp-sdk/simple-json)

#### LOOM

Unity的多线程插件。

### 客户端代码组织结构

客户端代码在Github上地址：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)

在Unity项目中的Assets目录下新建相关目录：

- Config: 存放一些游戏相关配置的json文件
- Plugins: 存放我们所需的插件；
- Scenes: 存放我们开发中需要保存的场景；
- Scripts: 存放我们客户端的所需脚本；
- Resources: 存放客户端用到的各种资源


我们把`pomelo-unityclient-socket`克隆到`Plugins`目录下，为了方便学习，这里我们不使用`.dll`文件，而采用原生的`cs`代码，同时删除不需要的测试代码。`SimpleJson`也是如此，只保留原生的`cs`代码。 具体的目录结构可以参照：[Github链接](https://github.com/vitahlin/VitahPomeloClient)


### 优化相关基础代码

为了代码更好的组织和扩展，我们新建若干类对协议进行封装和扩展。具体代码可以参看：`Assets/Scripts/Protocol/Core`。同时在`Scripts`目录下新建`Foundation/Util`目录，用于存放一些通用的工具类。

在客户端进行开发时，我们必然会涉及到UI表现，为此，我们还需要用一个类对UI界面进行管理。具体代码参看：`Assets/Scripts/Foundation/UIManager`

### 总结

至此，就大概完成了客户端开发环境的搭建，接下来我们只需要直接在此基础上进行游戏各个功能的开发。现阶段，我们不对客户端的相关代码进行分析，仅服务端 `Pomelo` 的使用介绍为主，后续如果有需要，我们会逐一分析客户端各个基础类的代码，以便了解底层的通信。