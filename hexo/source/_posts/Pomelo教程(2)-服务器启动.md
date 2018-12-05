---
title: Pomelo教程(2)-服务器启动
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: db8a7e3f
date: 2017-02-13 16:41:33
updated: 2017-02-13 16:41:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

上文中，介绍了 `Pomelo` 的安装，我们平常在开发时，会有多个部署版本，比如说开发服务器，以及正式的生产服务器。这里继续介绍**如何修改pomelo的运行环境配置**，使配置代码结构更清晰。

### 更改运行环境目录结构

我们可以看原来的服务器配置文件`game-server/config/servers.json`：
```javascript 
{
	"development": {
		"connector": [{
			"id": "connector-server-1",
			"host": "127.0.0.1",
			"port": 4010,
			"clientHost": "127.0.0.1",
			"clientPort": 3010,
			"frontend": true
		}]
	},
	"production": {
		"connector": [{
			"id": "connector-server-1",
			"host": "127.0.0.1",
			"port": 4010,
			"clientHost": "127.0.0.1",
			"clientPort": 3010,
			"frontend": true
		}]
	}
}
```

配置文件的格式是json，里面指定了开发环境development和生产环境production。当我们启动pomelo时，通过参数指定要运行的对应环境。这样的话，当环境增多以及服务器增多，文件内容就变得多而繁琐，我们可以设置不同的目录来区分不同的环境。

<!--more-->

在`game-server/config/`目录下新建`development_local`的环境，代表运行在本机的开发环境。同时，修改`servers.json`和`master.json`的文件内容。

同时我们需要修改`app.js`的文件内容，重新设定`connector`服务器的连接方式。原有配置：
```javascript
app.configure('production|development', 'connector', function() {}
```

我们需要改掉环境对应的名称，可以设定参数`all`，这样不管是什么环境都生效：

```javascript
app.configure('all', 'connector', function () {}
```

因为我们只在本地进行开发测试，所以建立一个环境即可，后续如果有需要我们继续修改对应配置即可。

### 服务器启动

进入到服务端的`game-server`目录下，最简单的，开发的时候直接`node app.js`就可以启动测试了，一般开发时不会配置很多个服务器，顶多就是在localhost多配置几个端口来模拟server，分布式部署时需要使用`pomelo start`来启动。
完整语法如下：
```c 
pomelo start [-e development | -e production] [--daemon]
```
默认启动的是`development`模式，如果要启动正式环境，需要额外加一个`production`参数，`--daemon`指的是后台运行，如果没有这个参数，断开SSH连接时项目也会自动停止，如果想要它持久的在后台运行，必须加这个参数，正式环境下一般启动命令如下：
```c 
pomelo start -e production --daemon
```
如果有设置在后台运行，那么停止时需要使用`pomelo stop`命令，不然的话直接`Ctrl+c`或者直接关闭命令窗口即可。

这里我们在`game-server`新建一个启动脚本`begin.sh`以启动服务器，每次启动运行脚本即可，而不用输入命令，脚本内容：

```c 
#!/bin/sh
pomelo start -e development_local
```

保存后，每次在终端中运行 `sh begin.sh`即可启动服务器。  


### 参考链接
- app.js配置参考 [https://github.com/NetEase/pomelo/wiki/%E6%B8%B8%E6%88%8F%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9A%84app.js%E9%85%8D%E7%BD%AE%E5%8F%82%E8%80%83](https://github.com/NetEase/pomelo/wiki/%E6%B8%B8%E6%88%8F%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9A%84app.js%E9%85%8D%E7%BD%AE%E5%8F%82%E8%80%83)