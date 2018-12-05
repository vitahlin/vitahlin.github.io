---
title: Pomelo教程(6)-客户端建立与服务端的连接
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: 99fd6935
date: 2017-02-25 23:04:33
updated: 2017-02-25 23:04:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

当我们搭建完服务端和客户端开发环境，就可以在此基础上进行游戏中各个功能的开发。这里，我们先尝试着建立客户端和服务端的连接。

<!--more-->

### 服务端协议格式

我们可以查看服务端的代码结构，`game-server/app`目录下只有一个`servers`文件夹，这个文件夹用来存放各个服务器逻辑的代码，可以在命令行中查看该目录的树形结构
```c 
.
└── servers
    └── connector
        └── handler
            └── entryHandler.js
```

可以看到，初始化服务端代码后，`servers`目录下只有一个`connector`服务器，该服务器下`handler`目录下只有一个`entryHandler.js`文件。

> 用Pomelo实现的服务器的对外接口只有两类，一类是接收客户端的请求，叫做handler，一类是接收RPC请求，叫做remote。我们只需要利用目录结构与服务器对应的形式，就可以快速实现服务器的抽象。

我们接着来看`entryHandler.js`的代码：
```javascript
Handler.prototype.entry = function(msg, session, next) {
	next(null, {
		code: 200,
		msg: 'game server is ok.'
	});
};

Handler.prototype.publish = function(msg, session, next) {
	var result = {
		topic: 'publish',
		payload: JSON.stringify({
			code: 200,
			msg: 'publish message is ok.'
		})
	};
	next(null, result);
};

Handler.prototype.subscribe = function(msg, session, next) {
	var result = {
		topic: 'subscribe',
		payload: JSON.stringify({
			code: 200,
			msg: 'subscribe message is ok.'
		})
	};
	next(null, result);
};
```

我们可以看到这里定义三个方法，分别是`entry`、`publish`和`subscribe`。那么，因为`entryHandler.js`属于`handler`文件夹，所以这三个方法就是用于接收客户端请求的。


### 客户端编写连接协议

#### 协议格式 

知道了服务端制定的协议格式，我们就可以根据该协议来进行通信。

在客户端的`Assets/Scripts/Protocol/`目录下新建`Connector`目录，以代表`connector`服务器的相关协议，在`Connector`目录下新建`ConnectorEntryRequest.cs`脚本：
```c 
using SimpleJson;
using System;
using Pomelo.DotNetClient;
using UnityEngine;

// 连接协议
public class ConnectorEntryRequest : BaseRequest
{
    public void Entry()
    {
        JsonObject msg = new JsonObject();
        this.Request(ProtocolRoute.CONNECTOR_ENTRY, msg);
    }
}
```

我们在`ProtocolRoute`类中对`CONNECTOR_ENTRY`的定义是：
```c 
public static string CONNECTOR_ENTRY = "connector.entryHandler.entry";
```

看到这里，我们就可以明白客户端与服务端制定协议名称的规则，即 **服务器名.文件名.方法名**。

完成后，我们新建登录场景和登陆界面来测试是否能正常连接。

#### 建立登陆场景 

新建`App.cs`挂载在`Login`场景的`Main Camera`上：
```c
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class App : MonoBehaviour {
	void Start () {

		// 初始化loom
		Loom.CreateThreadPoolScheduler();

		// 载入登陆窗口
		UIManager.Instance.PushWindow(UIWindowDefine.LoginWindow);
	}
}
```
即一进入登陆场景就载入登陆窗口。

#### 创建登陆窗口 

创建登陆窗口预设，放置于目录`Assets/Resources/Prefabs/UI/UILoginWindow`，新建脚本`UILoginWindow.cs`用于实现登陆窗口的各种逻辑，当点击登陆窗口中的登陆按钮时，就调用连接协议尝试与服务端进行连接。

#### 服务端与客户端连接方式

当我们点击Login按钮时，却打印出`OnDisconnect`字样，表示连接失败。
![](http://oc1mf55gf.bkt.clouddn.com/pomelo20170208001.png)

> 在与客户端通信的时候，pomelo目前提供了hybridconnector和sioconnector，其中hybridconnector支持tcp，websocket；sioconnector支持socket.io。

我们可以查看服务端的`app.js`代码，可以看到如下这段代码：
```javascript
app.configure('production|development', 'connector', function() {
	app.set('connectorConfig', {
		connector: pomelo.connectors.sioconnector,
		//websocket, htmlfile, xhr-polling, jsonp-polling, flashsocket
		transports: ['websocket'],
		heartbeats: true,
		closeTimeout: 60,
		heartbeatTimeout: 60,
		heartbeatInterval: 25
	});
});
```

说明pomelo默认的是`sioconnector`方式，而客户端采用的是socket，所以我们需要修改这部分的配置，将这段代码注释，改为如下代码：
```javascript 
app.configure('production|development', 'connector', function() {
	app.set('connectorConfig', {
		connector: pomelo.connectors.hybridconnector,
		heartbeat: 30
	});
});
```

同时，客户端与服务端的前端服务器`connector`相连，服务端中服务器的配置应该设置对应参数。查看服务器配置文件：`game-server/config/servers.json`中的connector配置，注意：
```javascript 
"clientPort": 3010,   // 客户端连接时监听的端口
 "frontend": true， 　// 前端服务器
```

客户端端口定义文件`Assets/Scripts/Protocol/Core/ServerHost.cs`中定义的端口应该和服务端设定的一样。

保存后，重启服务器。

这时候，我们再点击登陆按钮进行测试，可以看到如下效果，可以看到客户端打印出连接正常的信息：
![](http://oc1mf55gf.bkt.clouddn.com/pomelo0208002.png)

可以发现，`msg` 字段中的内容正是 `entryHandler.js` 的 `entry` 方法中返回的 `msg` 内容，这里仅测试 `entry` 方法，有需要还可以自行测试 `publish` 或者 `subscribe` 方法。

### 参考链接

- Pomelo Connector实现 [https://github.com/NetEase/pomelo/wiki/Connector%E5%AE%9E%E7%8E%B0](https://github.com/NetEase/pomelo/wiki/Connector%E5%AE%9E%E7%8E%B0)
- 前端服务器配置 [http://blog.csdn.net/nynyvkhhiiii/article/details/50352789](http://blog.csdn.net/nynyvkhhiiii/article/details/50352789)
- hybridconnector相关配置 [http://blog.csdn.net/wangqiuyun/article/details/12191121](http://blog.csdn.net/wangqiuyun/article/details/12191121)
- Pomelo服务器启动 [http://mygit.me/2016/07/15/pomelo/](http://mygit.me/2016/07/15/pomelo/)