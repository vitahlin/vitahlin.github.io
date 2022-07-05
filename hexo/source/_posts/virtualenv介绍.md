---
title: virtualenv介绍
tags: Python
categories: Python
comments: true
abbrlink: 7706752f
date: 2016-11-24 19:15:33
updated: 2016-11-24 19:15:33
copyright: true
---

在开发Python应用程序的时候，系统安装的Python只有一个版本。所有第三方的包都会被pip安装到同一个目录下。
如果我们要同时开发多个应用程序，那这些应用程序都会共用一个Python，但是需要的模块不同，如果应用A需要jinja 2.7，而应用B需要jinja 2.6怎么办？
这种情况下，每个应用可能需要各自拥有一套“独立”的Python运行环境。这时候我们就需要用到virtualenv。

virtualenv用于创建独立的Python开发环境，多个环境相互独立，互不影响，它能够：
1. 在没有权限的情况下安装新套件
2. 不同应用可以使用不同的套件版本
3. 套件升级不影响其他应用

<!--more-->

virtualenv是如何创建“独立”的Python运行环境的呢？原理很简单，就是把系统Python复制一份到virtualenv的环境，用命令`source env/bin/activate`进入一个virtualenv环境时，virtualenv会修改相关环境变量，让命令python和pip均指向当前的virtualenv环境。

### 用法

 - 安装
 ```c
 pip install virtualenv
 ```

- 创建环境
```c
virtualenv env
```
用命令创建一个名为**env**一个独立的虚拟环境，我们也可以加上参数`--no-site-packages`，这样已经安装到系统Python环境中的所有第三方包都不会复制过来，就得到了一个不带任何第三方包的“干净”的Python运行环境，形如：`virtualenv --no-site-packages env`

- 启动 
```c
cd env
source ./bin/activate
```
新建的Python环境被放到当前目录下的**env**目录。有了**env**这个Python环境，可以用source进入该环境。
用`source`命令进入环境后，命令行里面的提示会有变化，有个`(env)`前缀，表示当前环境是一个名为`env`的Python环境。

- 安装模块或者运行Python
如：
```c 
pip install jinja2
python test.py
```
在env环境下，用pip安装的包都被安装到env这个环境下，系统Python环境不受任何影响。也就是说，env环境是专门针对某个应用创建的。

- 退出环境

```c 
deactivate
```
运行后可以看到命令提示符没有**(env)**前缀，此时就回到了正常的环境，现在pip或python均是在系统Python环境下执行。

- 删除

删除环境的话直接删除**env**目录。

- 生成环境依赖

生成环境依赖，可以生成类似Node.js项目中的`package.json`文件，下次可以根据文件直接安装该项目所需的依赖。

相关命令
```c
pip freeze  #显示当前环境的所有依赖
pip freeze > requirement.txt  // 生成requirement.txt文件
pip install -r requirement.txt  // 根据requirement.txt生成相同的环境
```

**注：新环境的Python版本需要一致，不然可能出错。**