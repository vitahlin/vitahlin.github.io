---
title: Pomelo源代码分析(2)-拷贝目录
tags:
  - Pomelo
  - Node.js
categories: Pomelo
comments: true
copyright: true
abbrlink: cae33aa
date: 2017-06-19 16:57:33
updated: 2017-06-19 16:57:33
---

**代码复制时间：2017年4月5日  源代码版本：5f7bec1**

**GitHub项目链接：[https://github.com/vitahlin/pomelo-analyze](https://github.com/vitahlin/pomelo-analyze)**

---

### 源代码

`/bin/pomelo`中的拷贝目录函数：

```JavaScript
var fs = require('fs');
var path = require('path');
var mkdirp = require('mkdirp');

var FILEREAD_ERROR = 'Fail to read the file, please check if the application is started legally.';

function abort(str) {
	console.error(str);
	process.exit(1);
}

function mkdir(path, fn) {
	mkdirp(path, 0755,
		function(err) {
			if (err) {
				throw err;
			}
			console.log('   create : ' + path);
			if (typeof fn === 'function') {
				fn();
			}
		}
	);
}

function copy(origin, target) {
	if (!fs.existsSync(origin)) {
		abort(origin + 'does not exist.');
	}

	if (!fs.existsSync(target)) {
		mkdir(target);
		console.log('   create : '.green + target);
	}

	fs.readdir(origin,
		function (err, datalist) {
			if (err) {
				abort(FILEREAD_ERROR);
			}
			for (var i = 0; i < datalist.length; i++) {
				var oCurrent = path.resolve(origin, datalist[i]);
				var tCurrent = path.resolve(target, datalist[i]);
				if (fs.statSync(oCurrent).isFile()) {
					fs.writeFileSync(tCurrent, fs.readFileSync(oCurrent, ''), '');
					console.log('   create : '.green + tCurrent);
				} else if (fs.statSync(oCurrent).isDirectory()) {
					copy(oCurrent, tCurrent);
				}
			}
		}
	);
}
```

<!--more-->

### 代码分析

#### 判断目录是否正确

`fs.existsSync` 即同步版的 `fs.exists()`。用来判断文件路径是否存在。

```JavaScript
if (!fs.existsSync(origin)) {
		abort(origin + 'does not exist.');
	}

	if (!fs.existsSync(target)) {
		mkdir(target);
		console.log('   create : '.green + target);
	}
```
先判断，要拷贝的源目录是否存在，如果不存在的报错，并退出。

然后判断目的目录是否存在，如果不存在的话，先调用`mkdir`函数创建目录。

#### 拷贝目录下文件

```JavaScript
fs.readdir(origin,
		function (err, datalist) {
			if (err) {
				abort(FILEREAD_ERROR);
			}
			for (var i = 0; i < datalist.length; i++) {
				var oCurrent = path.resolve(origin, datalist[i]);
				var tCurrent = path.resolve(target, datalist[i]);
				if (fs.statSync(oCurrent).isFile()) {
					fs.writeFileSync(tCurrent, fs.readFileSync(oCurrent, ''), '');
					console.log('   create : '.green + tCurrent);
				} else if (fs.statSync(oCurrent).isDirectory()) {
					copy(oCurrent, tCurrent);
				}
			}
		}
	);
```

目录判断完成后，通过 `fs.readdir` 函数读取源目录下的文件信息，然后遍历文件信息。判断每个文件信息的内容，如果是文件，那么直接拷贝并写入到目标文件。如果是目录，那么递归调用`copy`函数。











 