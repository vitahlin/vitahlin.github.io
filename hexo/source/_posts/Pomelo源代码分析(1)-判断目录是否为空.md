---
title: Pomelo源代码分析(1)-判断目录是否为空
tags:
  - Pomelo
  - Node.js
categories: Pomelo
comments: true
copyright: true
abbrlink: f2e61d6f
date: 2017-06-15 15:57:33
updated: 2017-06-15 15:57:33
---

**代码复制时间：2017年4月5日  源代码版本：5f7bec1**

**GitHub项目链接：[https://github.com/vitahlin/pomelo-analyze](https://github.com/vitahlin/pomelo-analyze)**

---

从这篇文章开始，我们逐渐分析 `pomelo` 框架的源代码，因为自身水平限制，不能一下就读透 `pomelo` 的核心及其框架理念，因此，会从零开始，一点一点逐步读懂、注释，在此做记录，仅供参考。

项目目录：
```c
├── LICENSE
├── README.md
├── bin
├── gruntfile.js
├── index.js
├── lib
├── myTest
├── node_modules
├── package.json
├── template
└── test
```

如上述所示，`bin` 目录下存放用于设定 `pomelo` 相关命令的代码。`lib` 目录下下则是 `pomelo` 框架的核心代码。 `myTest`存放我在分析过程中的测试代码。`template`存放`pomelo`的模版代码。

为此，我们先从`pomelo`命令开始。

<!--more-->

### 源代码

`/bin/pomelo`中的判断目录是否为空函数：

```JavaScript
var fs = require('fs');
var FILEREAD_ERROR = 'Fail to read the file, please check if the application is started legally.';

function abort(str) {
	console.error(str);
	process.exit(1);
}

function emptyDirectory(path, fn) {
	fs.readdir(path, function (err, files) {
		if (err && 'ENOENT' !== err.code) {
			abort(FILEREAD_ERROR);
		}

	    fn(!files || !files.length);
	});
}
```

### 代码分析

`fs.readdir` 这个方法以异步的方式读取文件目录。

语法：
```JavaScript
fs.readdir(path, [callback(err,files)])
```

其中，`path`是目录路径，`callbask`是回调，传递两个参数 `err` 和 `files`，`files` 是一个包含 **指定目录下所有文件名称的数组**。

读取过程中如果目录不存在，会抛出异常，错误码是 `ENOENT` 。
如果目录存在，则判断参数`files`，当参数`files`不为空并且长度不为0，则表示目录不为空，返回`TRUE`，否则返回`FALSE`。

### 参考链接
- [http://www.runoob.com/nodejs/nodejs-fs.html](http://www.runoob.com/nodejs/nodejs-fs.html)
