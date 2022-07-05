---
title: Pomelo教程(5)-使用Grunt对服务端代码进行格式化
tags:
  - Pomelo
  - Node.js
  - Unity3D
categories: Pomelo
comments: true
copyright: true
abbrlink: pomelo_series_use_grunt_to_format
date: 2017-02-19 17:00:33
updated: 2017-02-19 17:00:33
---

**系统版本：macOS 10.12.3，Pomelo版本：1.2.3，Unity版本：5.4.4f1**

**Pomelo Demo 服务端代码：[https://github.com/vitahlin/VitahPomeloServer](https://github.com/vitahlin/VitahPomeloServer)**

**Pomelo Demo 客户端代码：[https://github.com/vitahlin/VitahPomeloClient](https://github.com/vitahlin/VitahPomeloClient)**

### 为什么需要Grunt

对于需要反复重复的任务，例如压缩、编译、单元测试、代码检查等，自动化工具可以减轻你的劳动，简化你的工作。这边我们可以用 `Grunt`来进行代码检查和格式化。

### Grunt安装

Grunt基于Node.js，安装之前要先安装Node.js。至于如何安装 `Node.js`，可以参考这个链接：[http://www.vitah.net/posts/51475956/](http://www.vitah.net/posts/51475956/)

`Node.js` 安装完成后，执行如下命令：

```c 
npm install -g grunt-cli
```

安装 `grunt-cli` 并不等于安装了 `Grunt` 任务运行器！`Grunt CLI` 的任务是运行 `Gruntfile` 指定的 `Grunt` 版本。 这样就可以在一台电脑上同时安装多个版本的 `Grunt`。还是没读懂到底`grunt-cli`和`grunt` 什么区别，那么你先使用吧，使用后你可以就明白了。

<!--more-->

### 在项目目录中安装grunt

在项目目录下执行如下命令，用 `npm` 安装 `grunt`:

```c
npm install grunt --save-dev
```


### 插件的加载

介绍完 `Grunt` 的配置，我们需要了解下 `Grunt` 的插件。

在这个链接[https://gruntjs.com/plugins](https://gruntjs.com/plugins)可以找到所有的 `Grunt` 插件。你可以搜索对应的插件名称，查看该插件的具体使用。

比如：

- 插件`grunt-jsbeautifier`：[https://www.npmjs.com/package/grunt-jsbeautifier](https://www.npmjs.com/package/grunt-jsbeautifier)
- 插件`grunt-contrib-jshint`：[https://www.npmjs.com/package/grunt-contrib-jshint](https://www.npmjs.com/package/grunt-contrib-jshint)

在对应的链接里面，我们可以找到这个插件的安装方式及其配置方法。

安装完成后，在目录下增加`grunt-contrib-jshint`插件的配置文件，[`.jshintrc`](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/.jshintrc)
和 [`.jshintignore`](https://github.com/vitahlin/VitahPomeloServer/blob/master/game-server/.jshintignore)


### grunt相关配置

了解完 `Grunt` 的插件后，需要增加对应的`Gruntfile.js`配置，主要包含三块代码：**任务配置代码、插件加载代码、任务注册代码。**

#### 任务配置代码

例如下面代码：

```c 
grunt.initConfig({
		jsbeautifier: {
			files: ['./app/**/*.js', './config/**/*.json'],
			options: {
				js: {
					braceStyle: 'collapse',
					breakChainedMethods: false,
					e4x: true,
					evalCode: true,
					indentChar: ' ',
					indentLevel: 0,
					indentSize: 4,
					indentWithTabs: false,
					jslintHappy: true,
					keepArrayIndentation: true,
					keepFunctionIndentation: true,
					maxPreserveNewlines: 2,
					preserveNewlines: true,
					spaceBeforeConditional: true,
					spaceInParen: false,
					unescapeStrings: true,
					wrapLineLength: 0,
					endWithNewline: false
				}
			}
		},
		jshint: {
			files: [
				'./app/**/*.js',
				'./*.js'
			],
			options: {
				jshintrc: '.jshintrc'
			}
		}
	});
```

具体的任务配置代码以对象格式放在 `grunt.initConfig` 函数里面，至于怎么配置里面的参数内容，需要查看每个插件的用法，根据用法来编写任务。

#### 插件加载代码

写下下面代码即可加载：

```c 
grunt.loadNpmTasks('grunt-jsbeautifier');
grunt.loadNpmTasks('grunt-contrib-jshint');
```

因为现在这边只用到 `grunt-jsbeautifier` 和 `grunt-contrib-jshint` 插件，所以加载这两个就可以了。

#### 任务注册代码

插件也加载了，任务也布置了，下面我们得注册一下任务，使用

```c 
grunt.registerTask('default', ['jsbeautifier', 'jshint']);
```


### 运行`grunt`

按照上述步骤配置完成后，就可以在目录下运行命令`grunt`，运行结果如下所示：
![](http://oc1mf55gf.bkt.clouddn.com/pomelo20170419001.png)

出现`Done`字样说明文件已经检查过，没有出现基本错误并且已经格式化，如果提示出错，根据提示内容的文件和所在行数进行修改即可。

### 参考链接

- [https://www.npmjs.com/package/grunt-contrib-jshint](https://www.npmjs.com/package/grunt-contrib-jshint)
- [http://yujiangshui.com/grunt-basic-tutorial/](http://yujiangshui.com/grunt-basic-tutorial/)
- [https://gruntjs.com/](https://gruntjs.com/)
