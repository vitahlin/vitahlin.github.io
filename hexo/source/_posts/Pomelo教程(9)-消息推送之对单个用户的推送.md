---
title: Pomelo教程(9)-消息推送之对单个用户的推送
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: 9ed74384
date: 2017-03-21 15:07:33
updated: 2017-03-21 15:07:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

游戏中，用户经常会涉及到资源的操作，比如说收取资源，购买装备消耗资源，战斗消耗资源，战利品获取资源等，为了保证客户端和数据库数据的统一，我们不得不频繁的调用协议来获取用户的基本信息，这样就显得很是烦琐，所以我们可以用消息推送来做，当数据库里面的数据更新时，服务端主动推送新的资源信息给客户端，而不是由客户端来主动调用。

消息推送有两类，一类是对单个用户的推送，如这个用户的基本信息啊、装备信息等等；一类是全部用户的推送，如世界聊天，需要对全部用户推送这个内容，如工会聊天、工会信息需要对工会内的全部用户进行推送。这里介绍如果实现对单个用户的推送。

### pomelo-status-plugin
我们需要用到 pomelo-status-plugin ，它是一个全局的用户状态存储的插件，使用该插件可以对用户所在前端服务器进行查询，同时还能在全局范围内给具体用户进行消息推送。

<!--more-->

#### 安装pomelo-status-plugin

在工程目录下执行命令，安装模块：
```c 
npm install pomelo-status-plugin
```

#### 增加status的配置文件
新增`game-server/config/development_local/status.json`文件配置：
```c 
{
    "prefix": "status",
    "host": "127.0.0.1",
    "port": 6379,
    "cleanOnStartUp": true
}
```

#### 采用status配置

修改 `app.js` 文件，新增以下代码，采用配置:

```c 
var status = require('pomelo-status-plugin');

app.configure('all', function() {
	// pomelo-status-plugin配置
	var status_config = configUtil.load('status');
	app.use(status, {
		status: status_config
	});
});
```

### 服务端实现用户推送的相关代码

我们新建文件 `game-server/app/services/pushServices.js` 于来封装用户信息推送的函数：
```c 
var roleMgr = require('../mgr/roleMgr.js');
var jsonDiffUtil = require('../util/jsonDiffUtil.js');

exports.pushRoleModify = function (uid, old_json, new_json) {
	var params = jsonDiffUtil.getChangedJson(old_json, new_json);
	roleMgr.sendRoleChange(
		uid,
		params,
		function (err, fails) {
			if (err) {
				console.error('send role message error: %j, fail ids: %j', err, fails);
				return;
			}
		}
	);
};
```

其中 `uid` 是当前用户的 `uid` ,`old_json` 代表旧信息的json格式内容，`new_json` 代表最新信息等json格式内容，然后通过`jsonDiffUtil`简单计算信息的更新内容，来实现增量推送。

什么是增量推送？简单的说就是只推送更改的内容，比如说旧数据内容和新数据内容如下：
```c 
// 旧数据内容
{"uid":10001,"gold":100,"name":"testname","lv":1}

// 新数据内容
{"uid":10001,"diamond":1000,"gold":1001,"name":"testname","lv":1}
```

那么通过计算，得出数据的更改只有 `dismoand` (新增数据)和 `gold`（内容变化）,那么只需要推送 `diamond` 和 `gold` 即可，推送内容如下：
```c 
{"diamond":1000,"gold":1001}
```

在`mgrUtil.js`中封装了推送的基本函数，而后在`roleMgr.js`继续封装用于用户推送：
```c 
var eventType = require('../consts/eventType');
var mgrUtil = require('../util/mgrUtil');

/**
 * 用户信息推送
 * @param  {[type]}   uid    [用户id]
 * @param  {[type]}   params [推送内容]
 * @param  {Function} cb     [description]
 * @return {[type]}          [description]
 */
exports.sendRoleChange = function(uid, params, cb) {
	mgrUtil.sendStatusMessage(
		[uid],
		eventType.ON_ROLE_CHANGE,
		params,
		cb
	);
};
```

`eventType.js`定义了推送的定义等等，这里不在赘述，可以参看源代码。

### 客户端实现推送相关代码

在服务端完成了推送的功能以后，客户端需要实现推送的接收，以及对推送的监听。

新建 `Assets/Scripts/Push/ProtocolEvent.cs`，用来定义推送，需要注意的是这个值和服务端 `eventType.js` 定义的值要相同才行：
```c 
public static class ProtocolEvent
{
    // uset data push
    public static string ON_ROLE_CHANGE = "onRoleChange";
}
```

同样在 `Assets/Scripts/Push/` 完成相关需要的工具类，具体可以参看源代码，其中 `PushEventNotifyCenter` 用观察者模式实现对推送的监听和 UI 刷新。

然后我们在 Main 场景中创建一个 Text 用于显示用户的基本信息，并且实现一个添加资源协议，调用时增加用户的金币和钻石，来帮助更直观的展现推送的用法。

在 Main 场景中新建空物体，挂载 `MainSceneStart` 脚本用于在 Main 场景开始时，实现界面的载入。主场景界面代码 `UIMainSceneWindow`，实现了初次进入场景时显示用户信息，以及增加资源协议的调用，以及对推送的监听，具体可参看代码注释。

修改 `Login` 场景的 `UILoginWindow` 脚本，在登录成功时，实现推送的初始化
```c 
// 登陆数据获取成功
private void OnLoginDataSuccess()
{
	PushManager.Instance.Init();
	PushEventNotifyCenter.Instance.RemoveAllListener();		Loom.DispatchToMainThread(() => SceneManager.LoadScene("Main"));
}
```

完成后，我们运行 Demo，在每次点击按钮实现资源的增加时，客户端都会收到服务端的推送，并实现UI界面显示的资源数目的刷新。

### 参考链接
- pomelo 消息推送代码示例 [http://blog.csdn.net/linminqin/article/details/24548785](http://blog.csdn.net/linminqin/article/details/24548785)
- pomelo-status-plugin [https://github.com/NetEase/pomelo/wiki/Pomelo%E4%BD%BF%E7%94%A8%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C#pomelo-status-plugin](https://github.com/NetEase/pomelo/wiki/Pomelo%E4%BD%BF%E7%94%A8%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C#pomelo-status-plugin)