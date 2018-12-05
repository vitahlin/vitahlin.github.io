---
title: Node.js中的fs.readdir方法使用说明
tags: Node.js
categories: Node.js
comments: true
copyright: true
abbrlink: function_fs.readdir
date: 2017-06-07 14:38:33
updated: 2017-06-07 14:38:33
---

### 方法说明

以异步的方式读取文件目录。

### 语法

代码如下：
```javascript
var fs= require('fs);
fs.readdir(path, [callback(err,files)])
```

使用前需要引入fs模块。

<!--more-->

接收参数说明：
- path: 目录路径
- callback: 回调，传递两个参数 err 和 files，files是一个包含 “ 指定目录下所有文件名称的” 数组。

### 参考代码

网易开源游戏服务器框架 `Pomelo` 里面的一个判断路径是否为空的函数：
```javascript
var fs = require('fs');

var FILEREAD_ERROR = 'Fail to read the file, please check if the application is started legally.';

function emptyDirectory(path, fn) {
	// fs.readdir函数读取对应的目录，传递两个参数 err 和 files，files是一个包含 “ 指定目录下所有文件名称的” 数组
	fs.readdir(path, function (err, files) {

		// 报错"ENORNT"，表示路径不存在，调用退出函数
		if (err && 'ENOENT' !== err.code) {
			abort(FILEREAD_ERROR);
		}

		// 路径读取正常，判断files是否为空或者数组长度为0
		// 如果为空返回 true，否则返回 false
		fn(!files || !files.length);
	});
}

function abort(str) {
	console.error(str);
	process.exit(1);
}
```