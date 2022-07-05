---
title: Pomelo源代码分析(3)-pomelo初始化命令分析
tags:
  - Pomelo
  - Node.js
categories: Pomelo
comments: true
copyright: true
abbrlink: '93499971'
date: 2017-06-22 16:57:33
updated: 2017-06-22 16:57:33
---

**代码复制时间：2017年4月5日  源代码版本：5f7bec1**

**GitHub项目链接：[https://github.com/vitahlin/pomelo-analyze](https://github.com/vitahlin/pomelo-analyze)**

---

前两篇中，我们看完了判断目录是否为空和拷贝目录的代码，接下来，我们就可以分析整个`pomelo`项目初始化的过程。

`pomelo`项目初始化的代码实现在文件`bin/pomelo`下面，内容太多，这里就不贴出来，可以查看链接：[https://github.com/vitahlin/pomelo-analyze/blob/master/bin/pomelo](https://github.com/vitahlin/pomelo-analyze/blob/master/bin/pomelo)

接下来，就可以来分析`pomelo`初始化的过程。

<!--more-->

### 用commander实现初始化命令

源代码：
```javascript
var program = require('commander');
program.command('init [path]')
	.description('create a new application')
	.action(function (path) {
		init(path || CUR_DIR);
	});
```

上述代码，引入`commander`模块，来定义`pomelo init`的命令，命令还有一个可选参数`path`，用于自行指定要初始化的目录，如果不指定，那么就默认传入当前路径`CUR_DIR`，命令执行完调用`init`函数。

### 初始化函数

源代码：
```javascript
function init(path) {
	console.log(INIT_PROJ_NOTICE);

	connectorType(function (type) {
		emptyDirectory(path, function (empty) {
			if (empty) {
				process.stdin.destroy();
				createApplicationAt(path, type);
			} else {
				confirm('Destination is not empty, continue? (y/n) [no] ', function (force) {
					process.stdin.destroy();
					if (force) {
						createApplicationAt(path, type);
					} else {
						abort('Fail to init a project'.red);
					}
				});
			}
		});
	});
}
```

上述初始化函数中，先打印一条提示内容，然后调用`connectorType`函数，来让用户指定项目的网络连接类型，设定完连接类型后，调用`emptyDirectory`函数判断要初始化项目的位置是不是空目录，不是空的目录的话提示用户进行清空目录操作，然后调用`createApplicationAt`函数进行项目初始化。

### 用户指定网络连接类型
源代码：
```javascript
function connectorType(cb) {
	prompt('Please select underly connector, 1 for websocket(native socket), 2 for socket.io, 3 for wss, 4 for socket.io(wss), 5 for udp, 6 for mqtt: [1]',
		function (msg) {
			switch (msg.trim()) {
				case '':
					cb(1);
					break;
				case '1':
				case '2':
				case '3':
				case '4':
				case '5':
				case '6':
					cb(msg.trim());
					break;
				default:
					console.log('Invalid choice! Please input 1 - 5.'.red + '\n');
					connectorType(cb);
					break;
			}
		}
	);
}
```

指定项目连接类型代码较简单，就是对用户的输入信息进行处理，msg是用户输入信息，用trim函数进行处理，去除字符串两端空白字符，然后对输入的字符串进行判断当输入内容为空，用户直接输入Enter键，返回1，当用户输入不为空，且输入正确，返回用户选择的连接类型，当输入有误，提示错误内容，并且再次让用户选择连接方式。

### 创建初始化项目所需的必备代码
源代码：
```javascript
function createApplicationAt(ph, type) {
	var name = path.basename(path.resolve(CUR_DIR, ph));

	// 复制template模版下的代码到指定初始化项目到目录
	copy(path.join(__dirname, '../template/'), ph);

	// 创建两个目录
	mkdir(path.join(ph, 'game-server/logs'));
	mkdir(path.join(ph, 'shared'));

	// rmdir -r 定义删除目录的内部函数
	var rmdir = function (dir) {
		var list = fs.readdirSync(dir);
		for (var i = 0; i < list.length; i++) {
			var filename = path.join(dir, list[i]);
			var stat = fs.statSync(filename);

			if (filename === "." || filename === "..") {

			} else if (stat.isDirectory()) {
				// 如果是目录，递归调用rmdir
				rmdir(filename);
			} else {
				// 如果是文件，直接删除
				fs.unlinkSync(filename);
			}
		}

		// 删除目录，返回值为null 或 undefined则表示删除成功，否则将抛出异常
		fs.rmdirSync(dir);
	};

	setTimeout(function () {
		switch (type) {
			case '1':
				// use websocket 指定不需要的代码文件
				var unlinkFiles = ['game-server/app.js.sio',
					'game-server/app.js.wss',
					'game-server/app.js.mqtt',
					'game-server/app.js.sio.wss',
					'game-server/app.js.udp',
					'web-server/app.js.https',
					'web-server/public/index.html.sio',
					'web-server/public/js/lib/pomeloclient.js',
					'web-server/public/js/lib/pomeloclient.js.wss',
					'web-server/public/js/lib/build/build.js.wss',
					'web-server/public/js/lib/socket.io.js'
				];

				// 直接删除
				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				break;
			case '2':
				// use socket.io
				var unlinkFiles = ['game-server/app.js',
					'game-server/app.js.wss',
					'game-server/app.js.udp',
					'game-server/app.js.mqtt',
					'game-server/app.js.sio.wss',
					'web-server/app.js.https',
					'web-server/public/index.html',
					'web-server/public/js/lib/component.json',
					'web-server/public/js/lib/pomeloclient.js.wss'
				];

				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				// 修改文件名
				fs.renameSync(path.resolve(ph, 'game-server/app.js.sio'), path.resolve(ph, 'game-server/app.js'));
				fs.renameSync(path.resolve(ph, 'web-server/public/index.html.sio'), path.resolve(ph, 'web-server/public/index.html'));

				rmdir(path.resolve(ph, 'web-server/public/js/lib/build'));
				rmdir(path.resolve(ph, 'web-server/public/js/lib/local'));
				break;
			case '3':
				// use websocket wss
				var unlinkFiles = ['game-server/app.js.sio',
					'game-server/app.js',
					'game-server/app.js.udp',
					'game-server/app.js.sio.wss',
					'game-server/app.js.mqtt',
					'web-server/app.js',
					'web-server/public/index.html.sio',
					'web-server/public/js/lib/pomeloclient.js',
					'web-server/public/js/lib/pomeloclient.js.wss',
					'web-server/public/js/lib/build/build.js',
					'web-server/public/js/lib/socket.io.js'
				];

				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				fs.renameSync(
					path.resolve(ph, 'game-server/app.js.wss'),
					path.resolve(ph, 'game-server/app.js')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/app.js.https'),
					path.resolve(ph, 'web-server/app.js')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/public/js/lib/build/build.js.wss'),
					path.resolve(ph, 'web-server/public/js/lib/build/build.js')
				);
				break;
			case '4':
				// use socket.io wss
				var unlinkFiles = ['game-server/app.js.sio',
					'game-server/app.js',
					'game-server/app.js.udp',
					'game-server/app.js.wss',
					'game-server/app.js.mqtt',
					'web-server/app.js',
					'web-server/public/index.html',
					'web-server/public/js/lib/pomeloclient.js'
				];

				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				fs.renameSync(
					path.resolve(ph, 'game-server/app.js.sio.wss'),
					path.resolve(ph, 'game-server/app.js')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/app.js.https'),
					path.resolve(ph, 'web-server/app.js')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/public/index.html.sio'),
					path.resolve(ph, 'web-server/public/index.html')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/public/js/lib/pomeloclient.js.wss'),
					path.resolve(ph, 'web-server/public/js/lib/pomeloclient.js')
				);

				rmdir(path.resolve(ph, 'web-server/public/js/lib/build'));
				rmdir(path.resolve(ph, 'web-server/public/js/lib/local'));
				fs.unlinkSync(path.resolve(ph, 'web-server/public/js/lib/component.json'));
				break;
			case '5':
				// use socket.io wss
				var unlinkFiles = ['game-server/app.js.sio',
					'game-server/app.js',
					'game-server/app.js.wss',
					'game-server/app.js.mqtt',
					'game-server/app.js.sio.wss',
					'web-server/app.js.https',
					'web-server/public/index.html',
					'web-server/public/js/lib/component.json',
					'web-server/public/js/lib/pomeloclient.js.wss'
				];

				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				fs.renameSync(
					path.resolve(ph, 'game-server/app.js.udp'),
					path.resolve(ph, 'game-server/app.js')
				);

				rmdir(path.resolve(ph, 'web-server/public/js/lib/build'));
				rmdir(path.resolve(ph, 'web-server/public/js/lib/local'));
				break;
			case '6':
				// use socket.io
				var unlinkFiles = ['game-server/app.js',
					'game-server/app.js.wss',
					'game-server/app.js.udp',
					'game-server/app.js.sio',
					'game-server/app.js.sio.wss',
					'web-server/app.js.https',
					'web-server/public/index.html',
					'web-server/public/js/lib/component.json',
					'web-server/public/js/lib/pomeloclient.js.wss'
				];

				for (var i = 0; i < unlinkFiles.length; ++i) {
					fs.unlinkSync(path.resolve(ph, unlinkFiles[i]));
				}

				fs.renameSync(
					path.resolve(ph, 'game-server/app.js.mqtt'),
					path.resolve(ph, 'game-server/app.js')
				);

				fs.renameSync(
					path.resolve(ph, 'web-server/public/index.html.sio'),
					path.resolve(ph, 'web-server/public/index.html')
				);

				rmdir(path.resolve(ph, 'web-server/public/js/lib/build'));
				rmdir(path.resolve(ph, 'web-server/public/js/lib/local'));
				break;
		}

		var replaceFiles = ['game-server/app.js',
			'game-server/package.json',
			'web-server/package.json'
		];

		for (var j = 0; j < replaceFiles.length; j++) {
			var str = fs.readFileSync(path.resolve(ph, replaceFiles[j])).toString();
			fs.writeFileSync(path.resolve(ph, replaceFiles[j]), str.replace('$', name));
		}

		// 重写版本号
		var f = path.resolve(ph, 'game-server/package.json');
		var content = fs.readFileSync(f).toString();
		fs.writeFileSync(f, content.replace('#', version));
	}, TIME_INIT);
}
```

如上述代码所示，先在目录下拷贝全部的代码文件，然后根据选择的连接类型，删除不需要的文件，并且对一些文件名进行重新命名，包括对`package.json`文件内容的重新处理。

完成后，这样，整个初始化过程就完成了。

### 流程总结

整个流程即是：
1. 输入pomelo init命令进行项目初始化
2. 提示用户选择项目的网络连接类型
3. 清空初始化项目所在的目录
4. 拷贝全部代码到指定目录
5. 根据用户指定的连接类型删除不需要的代码文件，并且对package.json文件内容进行处理
6. 初始化完成