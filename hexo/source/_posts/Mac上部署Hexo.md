---
title: Mac上部署Hexo
tags: Blog
categories: Blog
comments: true
abbrlink: use_hexo_in_mac
date: 2016-11-24 16:16:33
updated: 2016-11-24 16:16:33
copyright: true
---

**系统版本macOS 10.12.1，node版本是v5.12.0**

---


### 安装`Hexo`

安装Hexo

```c
npm install hexo-cli -g
```

初始化

```c
hexo init Blog
```

初始化完成，**`Blog`目录就是博客的根目录，所有的操作都在里面进行**。

<!--more-->

### [`Github`](https://github.com/)仓库设置

注册账号，然后建立Repository，**仓库名必须为`your_user_name.github.io`**，这里我的用户名是`vitahlin`，则仓库名是`vitahlin.github.io`。


### 设置推送到`Github`

修改`Blog`目录下的`_config.yml`文件，底部的`deploy`项目，修改：
```c
deploy:
  type: git
  repo: git@github.com:user_name/user_name.github.io.git
  branch: master
```
修改后保存，然后执行命令`npm install hexo-deployer-git --save`，再执行配置命令`hexo deploy`就可以在浏览器中打开链接`http://user_name.github.io/` 就可以看到自己的博客了。

#### Git设置ssh方式

上面`repo`设置方式为`git`的话，需要为`Github`生成`SSH Key`。参考链接：[链接地址](https://help.github.com/articles/generating-an-ssh-key/)  进行设置，或者百度一下"git ssh"很多网友写的中文介绍。


### 设置自己的域名

在Hexo的source目录下添加CNAME文件，没有任何后缀，在文件中填写要绑定的域名，如`www.user_name.net`。
然后设置域名解析，添加解析设置，纪录类型为CNAME,主机纪录www，解析线路为默认，然后纪录值填写github项目名称，如`user_name.github.io`，保存。
在Hexo中重新部署生成即可用自己的域名访问博客了。

参考链接
- [https://coding.net/help/doc/pages/index.html](https://coding.net/help/doc/pages/index.html)

### 设置部署到[`Coding`](https://coding.net/user)

因为GitHub在海外，国内访问因为各种原因访问速度会很慢，这时候我们可以选择把Hexo也部署到Coding上面，然后国内的IP可以访问Coding，速度有很大提高。

在Coding上新建项目，项目名字要和用户名称一样。在Hexo的source目录下新建文件`Staticfile`，没有文件格式没有内容，是为了让Coding知道这是用于部署静态网站。

#### 设置部署公钥


#### 开启Page服务

项目-代码-Pages服务中选择开启，如果有域名的话设置对应的域名。

#### 修改Hexo设置

修改Hexo的设置，修改目录下的`_config.yml`文件中的`deploy`项，改为如下设置：

```c
deploy:
  type: git
  repo: 
    github: git@github.com:user_name/user_name.github.io.git,master
    coding: git@git.coding.net:user_name/user_name.git,master
```

地址的用户名改为自己的即可。
Coding的地址可以通过项目-代码查看。


#### 修改域名解析设置

原来解析到GitHub的解析线路改为海外。新增两条解析设置解析到Coding的。最后设置如下：

| 记录类型   |主机记录     | 解析线路 | 记录值               |
|-----------| -------   | -----  | ---------------------|
| CNAME     | *         | 默认    | user_name.coding.me  |
| CNAME     | www       | 默认    | user_name.coding.me  |
| CNAME     | www       | 海外    | user_name.github.io  |

**上述中的user_name改为自己的用户名即可。**


### 相关命令

每次部署可以按以下步骤来执行：

```c
hexo clean
hexo g
hexo deploy
```

一些常用命令：

```c
hexo new"postName" #新建文章
hexo new page"pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub
hexo help # 查看帮助
hexo version #查看Hexo的版本
```
