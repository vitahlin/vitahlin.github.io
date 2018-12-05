---
title: Pomelo教程(12)-定时任务之schedule
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: 4bc986e
date: 2017-05-10 12:18:33
updated: 2017-05-10 12:18:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

上一篇中，我们介绍了如何用 `cron` 来实现定时任务，这里我们介绍另外一种方法，可以用`pomelo-scheduler`来实现定时任务。

`pomelo-schedule`模块介绍：[https://www.npmjs.com/package/pomelo-scheduler](https://www.npmjs.com/package/pomelo-scheduler)

Github链接：[https://github.com/NetEase/pomelo-scheduler](https://github.com/NetEase/pomelo-scheduler)

链接中详细介绍了模块的安装和使用方法。

<!--more-->

### 安装
在目录下用`npm`安装`pomelo-scheduler`：
```c 
npm install pomelo-scheduler -save
```
### 用法

新建`game-server/app/lib/schedule/userAddDiamond.js`([链接](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/app/lib/schedule/userAddDiamond.js))来实现定时任务的具体逻辑，我们在每次定时任务执行时，给用户加上1单位的钻石。

新建`game-server/app/lib/schedule.js`([链接](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/app/lib/schedule.js))来配置定时任务的运行时间等：
```javascript
'use strict';

var schedule = require('pomelo-schedule');
var userAddDiamond = require('./schedule/userAddDiamond');

////////////////////////////////////////////////////////////////////

schedule.scheduleJob(
    '0 0/1 * * * *',
    userAddDiamond, {
        name: 'addDiamond'
    }
);
```

`schedule`是引入的模块，`0 0/1 * * * *`是设置定时任务执行的时间规则，跟上一篇中`cron`中对时间的设置规则是一样的，`userAddDiamond`是具体要执行的函数，`name`则对应这个任务的名称。

完成定时任务的逻辑和时间设置后，我们需要在程序的运行过程中引入定时任务的执行，修改`game-server/app.js`([链接](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/app.js))文件：

```javascript
// 定义服务端和客户端的连接方式
app.configure('all', 'connector', function() {
	app.set('connectorConfig', {
		connector: pomelo.connectors.hybridconnector,
		heartbeat: 30
	});

	// 引入定时任务
	require('./app/lib/schedule.js');
});
```

如代码所示，在其中引入定时任务的执行配置。

保存代码后，重启服务器，我们就可以在客户端看到用户每分钟的资源都会增加1个单位，金币是用`cron`实现，而钻石则是用`schedule`实现。

### `schedule`的更多介绍

在`schedule`中还可以设置定时任务的执行周期等参数，更多功能可以参照模块的官网介绍：[链接](https://www.npmjs.com/package/pomelo-scheduler)。这里不做详细说明。

### 参考链接
- [https://www.npmjs.com/package/pomelo-scheduler](https://www.npmjs.com/package/pomelo-scheduler)
- [https://github.com/NetEase/pomelo-scheduler](https://github.com/NetEase/pomelo-scheduler)

