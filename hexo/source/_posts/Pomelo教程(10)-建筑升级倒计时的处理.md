---
title: Pomelo教程(10)-建筑升级倒计时的处理
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: bc618efe
date: 2017-04-10 11:40:33
updated: 2017-04-10 11:40:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

除了基本的信息交互，我们在游戏中也会时常遇到有倒计时的功能。比如说，建筑的升级需要倒计时，技能的升级需要倒计时，装备的建造等等。

这里，我们以建筑升级为例来说明类似功能的实现。一般服务端只记录升级的结束时间，倒计时放在客户端来实现，当客户端倒计时结束的时候，通知服务端刷新。服务端每次取相关表的时候也对排程进行刷新。

<!--more-->

### 创建保存建筑数据的 schema 和 model 

#### 保存单个建筑信息

我们创建 `game-server/app/schemas/build/buildItemSchema.js` 用来保存单个建筑的信息：
```javascript
buildItemschema.methods.isUpgradeTimeUp = function() {
	if (moment() >= moment(this.up_end_time) && moment(this.up_end_time).unix() != moment(0).unix()) {
		return true;
	}

	return false;
};

/**
 * [isEndUpgrade 判断是否升级完成（加上5秒的误差时间）]
 * @return {Boolean} [description]
 */
buildItemschema.methods.isEndUpgrade = function() {
	if (moment().add(5, 's') >= moment(this.up_end_time) && moment(this.up_end_time).unix() != moment(0).unix()) {
		return true;
	}

	return false;
};
```

注意代码中的这两个函数，`isUpgradeTimeUp`用于在服务端进行判断倒计时是否结束，而`isEndUpgrade`则是客户端倒计时结束后，通知服务端这个建筑已经升级完成了，应该对它的等级进行升级，这时候服务端进行验证的时候加入少许的误差时间。


#### 保存用户的全部信息

然后创建 `game-server/app/schemas/buildSchema.js` 用来保存用户的全部建筑信息：
```javascript
var buildSchema = new mongoose.Schema({
	// 用户id
	uid: {
		type: Number,
		required: true,
		unique: true,
		index: true
	},

	// 建筑列表
	build_list: [buildItemSchema],

	build_max_id: {
		type: Number,
		default: 10000
	}
});

/**
 * 建筑自增id
 * @return {[type]} [description]
 */
buildingSchema.methods.genBuildId = function() {
	this.build_max_id++;
	return this.build_max_id;
};

/**
 * 添加建筑
 * @param {[type]} build_type [要添加的建筑类型]
 * @param {[type]} position   [建筑位置]
 * @param {[type]} rotation   [建筑旋转角度]
 */
buildingSchema.methods.addBuild = function(build_type) {
	var new_build_id = this.genBuildId();
	this.build_list.push({
		build_id: new_build_id,
		lv: 0,
		type: build_type
	});

	return new_build_id;
};
```

注意其中的 `build_list` 是一个列表，保存所有建筑。`build_max_id` 则是用于生成每个建筑独一无二的建筑 ID。

#### 在 buildModel.js 中对排程进行刷新 

然后我们在每次取相关表的时候需要对排程进行刷新
`game-server/app/model/buildModel.js`：

```javascript
module.exports.getByUid = function(uid, cb) {
	modelUtil.getByUid(Build, uid, function(err, build_model) {
		if (!!err) {
			console.error(err);
			return cb(err);
		}

		dealSchedule(build_model, function(err) {
			if (build_model.isModified()) {
				build_model.save(function() {
					cb(null, build_model);
				});
			} else {
				cb(null, build_model);
			}
		});
	});
};

/**
 * 处理建筑中的排程
 * @param  {[type]}   build_model [description]
 * @param  {Function} cb          [description]
 * @return {[type]}               [description]
 */
var dealSchedule = function(build_model, cb) {
	var build_change = false;
	var build_old_json = build_model.toJSON();
	_.each(build_model.getBuildList(), function(build_item) {
		if (build_item.isUpgradeTimeUp()) {
			build_change = true;
			build_item.upgrade();
		}
	});

	var uid = build_model.uid;
	if (build_change) {
			pushService.pushBuildModify(uid, build_old_json, build_model.toJSON());
	}

	cb();
};
```

注意上文中的代码：
```javascript
if (build_item.isUpgradeTimeUp()) {
    build_change = true;
    build_item.upgrade();
}
```
在每次取用户表的时候，取完都会对建筑列表进行遍历，如果其中有倒计时时间结束的，则升级该建筑。并且这个判断时间是否结束，不加入对误差的处理，因为这是在服务端自身进行判断的，客户端并没有对此进行倒计时。

### 实现升级和刷新相关协议

#### 升级协议 
升级协议`game-server/app/servers/build/handler/build/upgrade.js`
```javascript
var build_item = build_model.getBuild(build_id);
if (!build_item) {
	return next(null, {
		code: code.BUILD_NOT_EXIST
	});
}

// 判断建筑是否在升级
if (build_item.getUpRemainTime() > 0) {
	return next(null, {
		code: code.BUILD_IS_IN_UPGRADE
	});
} else {
	// 建筑不在升级中，并且升级结束时间不为0，表示当前建筑上次升级以及完成，升级该建筑
	if (moment(build_item.up_end_time).unix() != moment(0).unix()){
		build_item.upgrade();
	}
}

// 设置建筑升级结束时间
build_item.setUpEndTime(300);
```

升级时会优先判断该建筑是否存在，所以在建筑升级前，我们应该建造该建筑，可以新建建造协议来完成此功能。然后升级时，判断该建筑是否在升级中，是的话返回错误码，不是的话直接设置升级结束时间，数据下发给客户端，由客户端开始倒计时功能。


#### 建造协议 
建造协议`game-server/app/servers/build/handler/buildHandler.js`： 
```javascript
var new_build_id = build_model.addBuild(build_type);
var build = build_model.getBuild(new_build_id);
build.setUpEndTime(300);
```

注意这三行代码，其实也就是把建造这个过程看作是建筑从0级到1级的升级过程，可以到`BuildSchema.js`中参考函数的具体代码。

#### 倒计时刷新协议 

倒计时结束刷新协议`game-server/app/servers/build/handler/build/refresh.js`
```javascript
module.exports = function (msg, session, next) {
    var uid = session.uid;
    var build_id = msg.build_id;

    if (!uid || !build_id) {
        return next(null, {
            code: code.PARAM_ERROR
        });
    }

    var onRefresh = function* () {
        var model_change = false;
        var build_model = yield thunkify(buildModel.getByUid)(uid);
        var build_old_json = build_model.toJSON();

        var refresh_build = {};
        _.each(build_model.getBuildList(), function (build_item) {
            if (build_item.build_id == build_id) {
                if (build_item.isEndUpgrade()) {
                    model_change = true;
                    build_item.upgrade();
                    refresh_build = build_item.toJSON();
                }
            } else {
                if (build_item.isUpgradeTimeUp()) {
                    model_change = true;
                    build_item.upgrade();
                }
            }
        });

        yield build_model.save();

        if (model_change) {
            pushService.pushBuildModify(uid, build_old_json, build_model.toJSON());
        }

        return next(null, {
            code: code.OK,
            result: {
                build_info: refresh_build
            }
        });
    };

    var onError = function (err) {
        console.error(err);
        return next(null, {
            code: code.FAIL
        });
    };
    co(onRefresh).catch(onError);
};
```
倒计时放在客户端计算，在界面打开时应该一直处于倒计时状态，倒计时结束后，调用该协议将客户端认为升级结束的建筑 ID 发给服务端，服务端进行验证（验证的时候为了准确性应该允许些许时间误差），验证后成功，将该建筑升级，把新的建筑数据推送下去。

这样我们就完成了含有倒计时的整个的建筑的功能。类似技能的升级、技能点研发、装备的建造等等都可以按类似的套路来完成。可以在客户端中跳转到建筑界面进行功能的测试。