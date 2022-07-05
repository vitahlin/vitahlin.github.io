---
title: npm常用命令介绍
tags: Node.js
categories: Node.js
comments: true
copyright: true
abbrlink: common_command_in_npm
date: 2017-08-07 17:23:33
updated: 2017-08-07 17:23:33
---

### npm init 创建package.json文件

安装包的信息可保持到项目的package.json文件中，以便后续的其它的项目开发或者他人合作使用，也说package.json在项目中是必不可少的。

```c
npm init [-f|--force|-y|--yes]
```

加上参数 `-f` 则会快速创建 `package.json` 文件。

<!--more-->

### npm install 安装模块

语法：
```c
npm install (with no args, in package dir)
npm install [<@scope>/]<name>
npm install [<@scope>/]<name>@<tag>
npm install [<@scope>/]<name>@<version>
npm install [<@scope>/]<name>@<version range>
npm install <tarball file>
npm install <tarball url>
npm install <folder>

alias: npm i
common options: [-S|--save|-D|--save-dev|-O|--save-optional] [-E|--save-exact] [--dry-run]
```

#### 本地安装
```c
npm install gulp
```

#### 全局安装
使用 `-g` 或 `--global`：
```c
npm install gulp -g
```

#### -S, --save 安装包信息将加入到 `dependencies`（生产阶段的依赖)

```c
npm install gulp --save 或 npm install gulp -S
```

`package.json` 文件的 `dependencies` 字段：
```c
"dependencies": {
    "gulp": "^3.9.1"
}
```

#### -D, --save-dev 安装包信息将加入到devDependencies（开发阶段的依赖）

开发阶段一般使用它。

```c
npm install gulp --save-dev 或 npm install gulp -D
```

`package.json` 文件的 `devDependencies` 字段：
```c
"devDependencies": {
    "gulp": "^3.9.1"
}
```

#### -O, --save-optional 安装包信息将加入到optionalDependencies（可选阶段的依赖）

```c
npm install gulp --save-optional 或 npm install gulp -O
```

`package.json` 文件的 `optionalDependencies` 字段：
```c
"optionalDependencies": {
    "gulp": "^3.9.1"
}
```

#### -E, --save-exact 精确安装指定模块版本

```c
npm install gulp --save-exact 或 npm install gulp -E
```

输入命令 `npm install gulp -ES` ，留意 `package.json` 文件的 `dependencies` 字段，以看出版本号中的 `^` 消失了：
```c
"dependencies": {
    "gulp": "3.9.1"
}
```

### npm uninstall 卸载模块 
语法：
```c
npm uninstall [<@scope>/]<pkg>[@<version>]... [-S|--save|-D|--save-dev|-O|--save-optional]

aliases: remove, rm, r, un, unlink
```

如卸载开发版本的模块：
```c
npm uninstall gulp --save-dev
```