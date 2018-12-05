---
title: Pomelo教程(11)-定时任务之cron
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: 5adaeec5
date: 2017-05-09 15:54:33
updated: 2017-05-09 15:54:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

在实际开发中，除了倒计时功能，我们平常还会遇到定时任务，比如说每日任务在每天的0点进行刷新，为此，我们就需要用到 `pomelo` 的定时功能，这里介绍用 `cron` 实现定时功能。我们以定时给玩家增加1个金币为示例。

<!--more-->

### 新建定时逻辑

每分钟给用户增加资源，是对用户信息的操作，所以我们把这个定时任务放在用户服务器下。

创建文件`game-server/app/servers/user/cron/addResCrons.js`([链接](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/app/servers/user/cron/addResCron.js))，代码中，`Cron.prototype.addRes`函数就是实现具体的逻辑，我们给用户增加1个金币，并且推送新的用户信息。

### 增加配置

实现定时逻辑后，需要增加对应的配置来启动定时服务，新建`game-server/config/development_local/crons.json`([链接](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/config/development_local/crons.json))文件来配置，配置内容如下：

```c
{
	"user": [{
		"id": 1,
		"serverId": "user-server-1",
		"time": "0 0/1 * * * *",
		"action": "addResCron.addRes"
	}]
}
```


第二行的 `user` 要跟服务器名称一致，`id`是定时任务在具体服务器的唯一标识，且不能在同一服务器中重复，`serverId` 是一个可选字段，如果有写该字段则该任务只在该服务器下执行，如果没有该字段则该定时任务在所有同类服务器中执行，`action`是具体执行任务方法。`addResCron.addRes`则代表执行 `game-server/app/servers/user/cron/addResCron.js中`的`addRes`方法。

`time`是定时任务执行的具体时间，时间的定义跟`linux`的定时任务类似，一共包括7个字段，每个字段的具体定义如下：

```c
*     *     *     *   *    *        command to be executed  
-     -     -     -   -    -  
|     |     |     |   |    |  
|     |     |     |   |    +----- day of week (0 - 6) (Sunday=0)  
|     |     |     |   +------- month (0 - 11)  
|     |     |     +--------- day of month (1 - 31)  
|     |     +----------- hour (0 - 23)  
|     +------------- min (0 - 59)  
+------------- second (0 - 59)  
```

例如:"0 30 10 * * *"，这就代表每天10:30执行相应任务；"0 0/1 * * * *"表示每一分钟执行相应任务。

配置完成后，重启服务器，打开客户端，我们就会发现每一分钟，服务器都会定时执行任务，给用户的金币数目加上1.