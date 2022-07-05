---
title: npm使用技巧
tags: Node.js
categories: Node.js
comments: true
copyright: true
abbrlink: skill_in_nodejs
date: 2017-08-07 17:38:33
updated: 2017-08-07 17:38:33
---

### 获取帮助

通过命令获取帮助：
```c
npm help
```
也可以获取特定命令的帮助
```c
npm help <command>
npm <command> -h
```

<!--more-->

### npm命令自动完成

npm通过bash提供了命令自动完成功能
```c 
npm completion >> ~/.bashrc
//or Z shell
npm completion >> ~/.zshrc

source ~/.bashrc
```
然后在终端中输入 `npm ins`，然后按下 `tab` 键就会出现 `install`。


### 修改全局模块的权限

当你试图安装全部模块时，类 `Linux` 系统可能会抛出权限错误，可以在 `npm` 命令之前添加 `sudo` 来执行，但这是一个较危险的选择。一个更好的解决方式是改变 `npm` 默认的模块安装目录：
```c 
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
```

使用适当的文本编辑器将下面的一行添加到 `~/.bashrc` 或者 `~/.zshrc` 文件中：
```c
export PATH="$HOME/.npm-global/bin:$PATH"
```

重新加载配置文件(`source ~/.bashrc`)，然后重新安装npm到用户所属路径：
```c
npm install -g npm
```
这个命令也会更新npm。

### 更新npm

可以通过`npm -v`显示当前的版本，如果有需要，可以通过下面的命令更新npm：
```c
npm install -g npm
```
当 Node 的主版本 released 之后，你也可能需要重新构建 C++ 扩展：
```c
npm rebuild
```


### 管理模块

#### 模块安装
```c 
npm install express      # 本地安装
npm install express -g   # 全局安装
```

#### 模块更新
```c 
npm update express     # 更新当前目录下的模块
npm update express -g  # 更新全局模块
```

#### 搜索模块
```c 
npm search express
```

#### 模块查看

如果已经安装了一些模块，可以用命令查看：
```c
npm list
```


也可以限制输出的模块层级：
```c
npm list --depth=0
```


打开一个模块的主页：
```c
npm home <package>
```


同样，可以打开一个模块的 Github 仓库:
```c
npm repo <package>
```


或者它的文档：
```c
npm docs <package>
```


或者它的bug列表：
```c
npm bugs <package>
```

### 使用淘宝 `NPM` 镜像
国内因为网络原因直接使用 `npm` 的官方镜像有时候是非常慢的，可以使用使用淘宝 `NPM` 镜像。

```c 
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

这样就可以使用 cnpm 命令来安装模块了：
```c
cnpm install [name]
```
