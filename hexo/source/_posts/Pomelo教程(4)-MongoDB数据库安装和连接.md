---
title: Pomelo教程(4)-MongoDB数据库安装和连接
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: aad869d4
date: 2017-02-16 15:55:33
updated: 2017-02-16 15:55:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

### Mongodb


这里我们采用`Mongodb`数据库，因为Mongodb支持的数据结构比较松散，是类似json的bson格式，和Node.js搭配起来更友好一点。

关于更多的关于Mongodb数据的介绍可以查看官网：[链接](https://www.mongodb.com/)

#### Mongodb安装

macOS上我们可以直接用`brew`安装：(`brew`的安装可以参照[这里](http://www.vitah.net/posts/28edc106/))
```c 
brew install mongodb 
```
<!--more-->

安装完成后，可以用`mongod -v`查看安装的数据库版本：
```c 
2016-12-07T20:31:52.699+0800 I CONTROL  [initandlisten] MongoDB starting : pid=67806 port=27017 dbpath=/data/db 64-bit host=VitahdeMacBook-Pro.local
2016-12-07T20:31:52.699+0800 I CONTROL  [initandlisten] db version v3.2.10
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] git version: 79d9b3ab5ce20f51c272b4411202710a082d0317
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] OpenSSL version: OpenSSL 1.0.2j  26 Sep 2016
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] allocator: system
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] modules: none
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] build environment:
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten]     distarch: x86_64
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten]     target_arch: x86_64
2016-12-07T20:31:52.700+0800 I CONTROL  [initandlisten] options: { systemLog: { verbosity: 1 } }
2016-12-07T20:31:52.700+0800 D NETWORK  [initandlisten] fd limit hard:9223372036854775807 soft:7168 max conn: 5734
2016-12-07T20:31:52.700+0800 E NETWORK  [initandlisten] listen(): bind() failed errno:48 Address already in use for socket: 0.0.0.0:27017
2016-12-07T20:31:52.700+0800 E NETWORK  [initandlisten]   addr already in use
2016-12-07T20:31:52.701+0800 E STORAGE  [initandlisten] Failed to set up sockets during startup.
2016-12-07T20:31:52.701+0800 I CONTROL  [initandlisten] dbexit:  rc: 48
```
可以看到这里的版本是`3.2.10`，接下来的开发都将以这个版本为主。

#### 使用LaunchRocket启动Mongodb

LaunchRocket是一个管理brew安装的service的工具，安装之后可以看所有的service的运行状态。macOS上我们可以用`LaunchRocket`([Github地址](https://github.com/jimbojsb/launchrocket))来帮助管理安装的服务，而不用在终端里面输入start之类的命令。

通过brew安装LaunchRocket:
```c
brew cask install launchrocket
```
安装完后，我们在macOS的系统偏好设置里面可以看到`LaunchRocket`的图标，单击点开管理界面，通过**Scan Homebrew**按钮可以搜索电脑上已经安装的服务，然后就在可以通过`LaunchRocket`快速启动或者关闭该服务了。

### Robomongo

Robomongo 是一个基于 Shell 的跨平台开源 MongoDB 管理工具。我们可以在Robomongo界面中更直观的查看数据库、修改数据等。
更多的介绍可以查看官网：[链接](https://robomongo.org/)

#### Robomongo安装

可以直接用`brew`安装：
```c 
brew cask install robomongo
```

#### Robomongo连接MongoDB

可以参考这个链接：[http://jingyan.baidu.com/article/9113f81b011ee72b3214c78d.html](http://jingyan.baidu.com/article/9113f81b011ee72b3214c78d.html)

### mongoose安装

Node.js有针对MongoDB的数据库驱动：mongodb。可以使用`npm install mongodb`来安装。不过直接使用mongodb模块虽然强大而灵活，但有些繁琐，我们可以使用`mongoose`。`mongoose`构建在mongodb之上，提供了Schema、Model和Document对象，用起来更为方便。

我们可以用Schema对象定义文档的结构（类似表结构），可以定义字段和类型、唯一性、索引和验证。Model对象表示集合中的所有文档。Document对象作为集合中的单个文档的表示。mongoose还有Query和Aggregate对象，Query实现查询，Aggregate实现聚合。

关于这些的信息，可以看这里：[http://mongoosejs.com/docs/guide.html](http://mongoosejs.com/docs/guide.html)

安装，在目录下执行`npm install mongoose --save`即可。

### 建立连接

在开发环境目录`game-server/config/development_local/`下新建`mongoodb.js`，用于定义数据库的相关信息：
```javascript
{
	"database": "PomeloDemo",
	"host": "127.0.0.1",
	"options": {
		"pass": "",
		"server": {
			"auto_reconnect": true,
			"socketOptions": {
				"keepAlive": 1
			}
		},
		"user": ""
	},
	"port": 27017
}
```

为了便于获取这个配置，我们新建一个工具类`game-server/app/util/configUtil.js`用于获取相关配置。

之后我们需要在`app.js`文件中建立和数据库的连接，在`app.js`中增加如下代码以进行和数据库的连接：

```javascript
var mongoose = require('mongoose');
var configUtil = require('./app/util/configUtil');

app.configure('all', function() {
	// 载入mongodb数据库的配置
	var mongodb_config = configUtil.load('mongodb');

	// 使用mongoose连接mongodb
	var db = mongoose.connect(mongodb_config.host, mongodb_config.database, mongodb_config.port, mongodb_config.options).connection;

	// 当数据库连接失败和成功时的处理
	db.on('error', console.error.bind(console, 'connection error:'));
	db.once('open', function callback() {
		console.log('数据库连接成功');
	});

	// 监控请求响应时间，如果超时就给出警告
	app.filter(pomelo.timeout());
});
```

保存后，重新启动服务器，可以看到打印出**数据库连接成功**字样，说明连接成功：

```c 
[2017-02-14 15:37:25.096] [INFO] pomelo-admin - [/Users/vitah/Downloads/Code/VitahPomeloServer/game-server/node_modules/pomelo-admin/lib/consoleService.js] try to connect master: "connector", "127.0.0.1", 3005

[2017-02-14 15:37:25.118] [INFO] console - 数据库连接成功
```

但是，这时候我们采用`Robomongo`去连接数据库，看到并没有`PomeloDemo`这个数据库，因为数据库现在里面没有任何数据，下一篇我们将继续介绍`mongoose`的使用。

### 参考链接

- mongoose用法介绍 

[http://blog.csdn.net/foruok/article/details/47746057](http://blog.csdn.net/foruok/article/details/47746057)

- mongoose安装和使用 

[http://www.nodeclass.com/api/mongoose.html](http://www.nodeclass.com/api/mongoose.html)

- Pomelo app.js配置参考 
[https://github.com/NetEase/pomelo/wiki/%E6%B8%B8%E6%88%8F%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9A%84app.js%E9%85%8D%E7%BD%AE%E5%8F%82%E8%80%83](https://github.com/NetEase/pomelo/wiki/%E6%B8%B8%E6%88%8F%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9A%84app.js%E9%85%8D%E7%BD%AE%E5%8F%82%E8%80%83)

- Pomelo Filter介绍 

[http://www.aiuxian.com/article/p-2240028.html](http://www.aiuxian.com/article/p-2240028.html)
- mongoose.createConnection()和mongoose.connect()区别

[http://cnodejs.org/topic/5226a922552118f11a04028e](http://cnodejs.org/topic/5226a922552118f11a04028e)