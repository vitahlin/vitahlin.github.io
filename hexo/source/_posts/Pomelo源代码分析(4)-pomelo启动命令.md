---
title: Pomelo源代码分析(4)-pomelo启动命令
tags:
  - Pomelo
  - Node.js
categories: Pomelo
comments: true
copyright: true
abbrlink: ed0d237d
date: 2017-07-24 17:44:33
updated: 2017-07-24 17:44:33
---

**代码复制时间：2017年4月5日  源代码版本：5f7bec1**

**GitHub项目链接：[https://github.com/vitahlin/pomelo-analyze](https://github.com/vitahlin/pomelo-analyze)**

---

`pomelo` 启动命令的代码比较简单，就是一个函数，直接参照注释即可。

<!--more-->

```javascript
var spawn = require('child_process').spawn;
var DEFAULT_GAME_SERVER_DIR = CUR_DIR;
var DEFAULT_ENV = 'development';

/*
 * 设置开始命令
 */
program.command('start')
	.description('start the application')

	// 当没有指定启动环境是，启动默认环境
	.option('-e, --env <env>', 'the used environment', DEFAULT_ENV)
	.option('-D, --daemon', 'enable the daemon start')

	// 同理，没指定路径时，启动默认路径
	.option('-d, --directory, <directory>', 'the code directory', DEFAULT_GAME_SERVER_DIR)
	.option('-t, --type <server-type>,', 'start server type')
	.option('-i, --id <server-id>', 'start server id')
	.action(function (opts) {
		start(opts);
	});

function start(opts) {
	// 判断app.js文件是否存在
	var absScript = path.resolve(opts.directory, 'app.js');
	if (!fs.existsSync(absScript)) {
		abort(SCRIPT_NOT_FOUND);
	}

	// 判断log目录是否存在，不存在，则新建log目录
	var logDir = path.resolve(opts.directory, 'logs');
	if (!fs.existsSync(logDir)) {
		fs.mkdir(logDir);
	}

	var ls;

	// 如果用户有指定服务器类型，那么采用指定的类型，否则选择全部
	var type = opts.type || constants.RESERVED.ALL;

	// 对参数进行处理
	var params = [absScript, 'env=' + opts.env, 'type=' + type];
	if (!!opts.id) {
		params.push('startId=' + opts.id);
	}

	// 判断是否选择后台启动，用child_pogress启动子进程
	if (opts.daemon) {

		// detached: 让子进程独立于父进程之外运行
		ls = spawn(process.execPath, params, {
			detached: true,
			stdio: 'ignore'
		});
		ls.unref();
		console.log(DAEMON_INFO);
		process.exit(0);
	} else {
		ls = spawn(process.execPath, params);
		ls.stdout.on('data', function (data) {
			console.log(data.toString());
		});
		ls.stderr.on('data', function (data) {
			console.log(data.toString());
		});
	}
}
```

`child_pogress` 的用法说明可以参照这里：[https://segmentfault.com/a/1190000007735211](https://segmentfault.com/a/1190000007735211)
