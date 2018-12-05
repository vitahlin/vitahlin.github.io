---
title: Pomelo教程(7)-新建用户服务器和用户信息表
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: dc1a186b
date: 2017-03-05 18:15:33
updated: 2017-03-05 18:15:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**


关于 Mongoose 的介绍网络上有很多，这里不再赘述。需要了解相关概念可以自行百度。

数据库安装完成后，我们需要往数据库里面保存相关数据，一个游戏里面玩家的用户数据是必不可少的，这里我们以此为例，介绍怎么使用 Mongoose 来进行数据的操作和存储。

### 创建 roleSchema.js  

在 Mongoose 中，所有东西都从一个 Schema 开始。每一个 schema 都映射到一个 MongoDb 的集合，并定义了该集合中的文档的形式。Schema 是一种以文件形式存储的数据库模型骨架，不具备数据库的操作能力，仅仅只是一段代码，无法通往数据库端, 仅仅只是数据库模型在程序片段中的一种表现。

<!--more-->

创建`game-server/app/schemas/roleSchems.js`用来定义玩家的信息结构：
```javascript
var mongoose = require('mongoose');

///////////////////////////////////////////////////////////

var roleSchema = new mongoose.Schema({
	// 角色id
	uid: {
		type: Number,
		required: true,
		unique: true,
		index: true
	},

	// 玩家等级
	lv: {
		type: Number,
		default: 1
	},

	// 玩家昵称
	name: {
		type: String,
		default: 'testname'
	},

	// 金币
	gold: {
		type: Number,
		default: 0
	},

	// 钻石
	diamond: {
		type: Number,
		default: 0
	}
});

/**
 * 获取用户id
 * @return {[type]} [description]
 */
roleSchema.methods.getUid = function() {
	return this.uid;
};

/**
 * 获取用户等级
 * @return {[type]} [description]
 */
roleSchema.methods.getLv = function() {
	return this.lv;
};

/**
 * 获取用户名称
 * @return {[type]} [description]
 */
roleSchema.methods.getName = function() {
	return this.name;
};

/**
 * 设置用户名字
 * @param {[type]} new_name [新的名字]
 */
roleSchema.methods.setName = function(new_name) {
	this.name = new_name;
};

/**
 * 获取金币
 * @return {[type]} [description]
 */
roleSchema.methods.getGold = function() {
	return this.gold;
};

/**
 * 添加金币
 */
roleSchema.methods.addGold = function(add_gold) {
	add_gold = parseInt(add_gold);
	if (add_gold >= 0) {
		this.gold += add_gold;
		return true;
	}

	return false;
};

/**
 * 扣除金币
 * @return {[type]} [description]
 */
roleSchema.methods.subGold = function(sub_gold) {
	sub_gold = parseInt(sub_gold);
	if (this.gold < sub_gold) {
		return false;
	}
	this.gold -= sub_gold;
	return true;
};

/**
 * 获取用户钻石
 * @return {[type]} [description]
 */
roleSchema.methods.getDiamond = function() {
	return this.diamond;
};

/**
 * 添加钻石
 */
roleSchema.methods.addDiamond = function(add_diamond) {
	add_diamond = parseInt(add_diamond);
	if (add_diamond >= 0) {
		this.diamond += add_diamond;
		return true;
	}
	return false;
};

/**
 * 扣除钻石
 * @return {[type]} [description]
 */
roleSchema.methods.subDiamond = function(sub_diamond) {
	sub_diamond = parseInt(sub_diamond);
	if (this.diamond < sub_diamond) {
		return false;
	}
	this.diamond -= sub_diamond;
	return true;
};

// 生成json格式内容的函数
if (!roleSchema.options.toJSON) {
	roleSchema.options.toJSON = {};
}
/* jshint unused:false */
roleSchema.options.toJSON.transform = function(doc, ret) {
	delete ret._id;
	delete ret.__v;
};

// 生成Role模型
mongoose.model('Role', roleSchema);
```

我们可以直接修改`roleSchema`文件来修改字段、类型、默认值等等，也可以自定义需要的函数来实现对字段的操作。

### 创建 roleModel.js 

接着创建`game-server/app/models/roleModel.js`用来实现增删改查等操作：

```javascript
var roleSchema = require('../schemas/roleSchema');
var mongoose = require('mongoose');
var Role = mongoose.model('Role');
var modelUtil = require('../util/modelUtil.js');

/////////////////////////////////////////////////////////////////////

/**
 * [getByUidNoCreate 获取玩家信息，不存在该玩家时不创建对应表]
 * @param  {[type]}   uid [玩家ID]
 * @param  {Function} cb  [回调]
 * @return {[type]}       [description]
 */
module.exports.getByUidNoCreate = function(uid, cb) {
	Role.findOne({
		uid: uid
	}, cb);
};

/**
 * [getByUid 获取玩家信息，不存在该玩家时创建对应表]
 * @param  {[type]}   uid [玩家ID]
 * @param  {Function} cb  [回调]
 * @return {[type]}       [description]
 */
module.exports.getByUid = function(uid, cb) {
	modelUtil.getByUid(Role, uid, function(err, role_model) {
		if (!!err) {
			console.error(err);
			return cb(err);
		}

		cb(null, role_model);
	});
};

```


注意代码：
```javascript
var roleSchema = require('../schemas/roleSchema');
var mongoose = require('mongoose');
var Role = mongoose.model('Role');
```
这三行用来和 `roleSchema` 相关联，是必不可少的。


然后我们封装了 `modelUtil.js` 工具类，当没有查找到对应ID的表的时候，直接创建表并返回。在 `roleModel` 里面定义了函数 `getByUid` 和 `getByUidNoCreate`，用来获取对应 `uid` 的用户数据。 

### 创建获取用户信息协议

完成表的创建后，我们需要通过协议对表进行操作以验证逻辑的正确性。

#### 新建用户服务器

我们可以新建一个用户服务器用来处理和用户相关的操作。

修改 `game-server/config/adminServer.js`，新增`user`的配置：
```javascript
[{
	"type": "connector",
	"token": "agarxhqb98rpajloaxn34ga8xrunpagkjwlaw3ruxnpaagl29w4rxn"
}, {
	"type": "user",
	"token": "agarxhqb98rpajloaxn34ga8xrunpagkjwlaw3ruxnpaagl29w4rxn"
}]
```

修改 `game-server/config/development_local/server.json`，新增`user`服务器的配置：
```javascript
{
	"connector": [{
		"id": "connector-server-1",
		"host": "127.0.0.1",
		"port": 4010,
		"clientPort": 3010,
		"frontend": true
	}],
	"user": [{
		"id": "user-server-1",
		"host": "127.0.0.1",
		"port": 4011
	}]
}
```

#### 服务端创建协议 

新建目录和代码文件 `game-server/app/server/user/handler/roleHandler.js`： 
```javascript
var co = require('co');
var thunkify = require('thunkify');
var roleModel = require('../../../models/roleModel.js');

// 错误码定义文件
var code = require('../../../consts/code.js');

module.exports = function(app) {
	return new Handler(app);
};

var Handler = function(app) {
	this.app = app;
};

/**
 * 获取玩家信息协议
 * @return {[type]} [description]
 */
Handler.prototype.getInfo = function(msg, session, next) {
	// 获取客户端传上来的UID 
	var uid = msg.uid;
	if (!uid) {
		return next(null, {
			code: code.PARAM_ERROR
		});
	}

	var onDo = function*() {
		// 获取对应Uid的model
		var role_model = yield thunkify(roleModel.getByUid)(uid);
		return next(null, {
			code: code.OK,
			result: {
				role_info: role_model.toJSON()
			}
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
```

`co` 和 `thunkify` 模块用来帮助实现 Node.js 的异步操作。关于它们的用法可以谷歌一下。

我们在协议中通过客户端传上来的 `uid` 获取对应的表，然后以 `json` 格式返回给客户端这个表的数据内容。 

#### 客户端调用协议获取信息

服务端新建协议后，我们就可以在客户端进行调用。

新建脚本 `UserRoleRequest.cs` 实现客户端协议的调用： 
```c 
// 获取用户信息协议
public class UserRoleRequest : BaseRequest
{
	public void getInfo()
	{
		// 定义我的Uid
		int my_uid = 10000;
		JsonObject msg = new JsonObject();
		msg.Add("uid", my_uid);
		this.Request(ProtocolRoute.USER_ROLE_GETINFO , msg);
	}
}
```

可以看到，里面我们定义了与服务端进行交互的 `uid`，一般游戏里面这个 `uid`是根据第一无二的设备号进行处理后得出的，或者接入第三方的账号，这里为了方便，我们直接在本地定义 `uid` 。 

同时，新建 `DataPool` 用于在游戏过程中数据的存储。

一般情况下，一登陆游戏，就需要得到用户的信息，而不是等跳转场景后再调用获取对应信息的协议，是故我们创建 `LoginData` 脚本用于游戏数据的初始化。在脚本中定义获取信息的协议，等协议全部成功后，再进行场景的跳转。 

修改 `UILoginWindow` 脚本内容：
```c 
private void OnEntry (RequestData data)
{
	Debug.Log ("OnEntry");
	Debug.Log (data);

	// 数据池清空
	DataPool.Destroy();

	// 获取相关登陆数据
	LoginData loginData = new LoginData();
	loginData.onSuccess = this.OnLoginDataSuccess;
	loginData.onError = this.OnLoginDataError;
	loginData.Receive();
}

// 登陆数据获取成功
private void OnLoginDataSuccess()
{
	Loom.DispatchToMainThread(() => SceneManager.LoadScene("Main"));
}

// 登陆数据获取失败
private void OnLoginDataError(RequestData data)
{
	this.loginBtn.gameObject.SetActive(true);
}
```

完成后，点击登陆按钮可以看到服务端返回回来的数据，并且跳转到 `Main` 场景。 我们可以通过 Robomongo 连接数据库，也可以看到这时候数据库已经不为空，并且存在对应的用户的数据表 `roles`。

完成这个协议后，你可以尝试下编写一个资源增加的协议，比如说每次客户端调用协议，都会给该用户增加100金币，我们也将会在后续用到这个协议。

### 参考链接
- mongoose 基本用法 [http://www.jianshu.com/p/9b20c1e2f373](http://www.jianshu.com/p/9b20c1e2f373)
- co thunkify 用法[https://gist.github.com/lleo/9928733](https://gist.github.com/lleo/9928733)
- 彻底理解 thunk 函数与 co 框架 [http://web.jobbole.com/85901/](http://web.jobbole.com/85901/)