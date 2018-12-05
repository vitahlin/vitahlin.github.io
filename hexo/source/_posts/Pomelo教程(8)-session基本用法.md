---
title: Pomelo教程(8)-session基本用法
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: pomelo_series_session_basic_usage
date: 2017-03-10 11:27:33
updated: 2017-03-10 11:27:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

上一篇中，我们已经建立了连接，并可以从服务端获取用户的基本数据，这时候我们如果同时开启两个客户端进行连接，会发现，两个客户端连接成功，并且处理的是同一个用户的数据。在很多情况下，我们应该保证连接的唯一性，跟你不能同时登录同一个微信账号一样。

我们需要用到 `pomelo` 的 `session` 来实现连接的管理。

### session 介绍

关于 session 可以参考 Github 上面的介绍：[链接地址](https://github.com/NetEase/pomelo/wiki/%E6%9C%AF%E8%AF%AD%E8%A7%A3%E9%87%8A#session-frontendsession-backendsession-sessionservice-backendsessionservice)

<!--more-->

### 修改服务端协议 

修改服务端连接协议，在协议中获取客户端传上来的 `uid`，然后和 `session` 绑定。

`game-server/app/servers/connector/handler/entryHandler.js`:
```c 
var co = require('co');
var thunkify = require('thunkify');
var pomelo = require('pomelo');
var code = require('../../../consts/code.js');

/////////////////////////////////////////////////////////////////////

module.exports = function(app) {
	return new Handler(app);
};

var Handler = function(app) {
	this.app = app;
};

Handler.prototype.entry = function(msg, session, next) {
	var self = this;
	var uid = msg.uid;
	if (!uid) {
		return next(null, {
			code: code.PARAM_ERROR
		});
	}

	var onDo = function*() {
		// 踢掉用户
		yield thunkify(pomelo.app.get('sessionService').kick)(uid);

		// session绑定uid
		yield thunkify(session.bind).bind(session)(uid);

		// 连接断开时的处理
		session.on('closed', onUserLeave.bind(self, self.app));

		// session同步，在改变session之后需要同步，以后的请求处理中就可以获取最新session
		yield thunkify(session.pushAll).bind(session)();

		return next(null, {
			code: code.OK
		});
	};

	var onError = function(err) {
		console.error(err);
		return next(null, {
			code: code.FAIL
		});
	};

	co(onDo).catch(onError);
};

// 断线之后的处理
var onUserLeave = function(app, session) {
	if (!session || !session.uid) {
		return;
	}

	console.log(session.uid + ' 已经断线');
};
```

同时，修改 `game-server/app/servers/user/handler/roleHandler.js` 的 `uid` 获取方式，改成直接从 `session` 中取得：
```c 
var uid = session.uid;
if (!uid) {
	return next(null, {
		code: code.PARAM_ERROR
	});
}
```

修改后，我们也需要修改客户端的相关协议。

### 修改客户端连接协议

之前，我们在`UserRoleRequest`中设置的用户的`uid`，我们改成直接在连接协议中`ConnectorEntryRequest`设置。

`UserRoleRequest.cs`:
```c 
public class UserRoleRequest : BaseRequest
{
	public void getInfo()
	{
		this.Request(ProtocolRoute.USER_ROLE_GETINFO, new JsonObject());
	}
}
```

`ConnectorEntryRequest.cs`:
```c 
public class ConnectorEntryRequest : BaseRequest
{
    public void Entry()
    {
		int my_uid = 10000;
        JsonObject msg = new JsonObject();
		msg.Add ("uid",my_uid);
        this.Request(ProtocolRoute.CONNECTOR_ENTRY, msg);
    }
}
```

完成后，重启服务器，这时候我们就会发现，同时只能登录一个账号，在新的客户端登录时，旧的必定会被顶掉，保证了连接的唯一性。

### 参考链接
- pomelo之session与sessionService分析 [http://blog.csdn.net/lisaem/article/details/49157119](http://blog.csdn.net/lisaem/article/details/49157119)
- session跟sessionService的区别 [http://nodejs.netease.com/topic/517791f5b5a2705b5a12555a](http://nodejs.netease.com/topic/517791f5b5a2705b5a12555a)
- pomelo源码分析 session组件 [http://imdiot.github.io/2014/08/13/pomelo-code-learn-2-component-session-2.html](http://imdiot.github.io/2014/08/13/pomelo-code-learn-2-component-session-2.html)
- 离开游戏的处理 [https://github.com/NetEase/pomelo/wiki/Treasure#5-leave-the-game](https://github.com/NetEase/pomelo/wiki/Treasure#5-leave-the-game)
- pomelo session 解释 [https://github.com/NetEase/pomelo/wiki/%E6%9C%AF%E8%AF%AD%E8%A7%A3%E9%87%8A#session-frontendsession-backendsession-sessionservice-backendsessionservice](https://github.com/NetEase/pomelo/wiki/%E6%9C%AF%E8%AF%AD%E8%A7%A3%E9%87%8A#session-frontendsession-backendsession-sessionservice-backendsessionservice)