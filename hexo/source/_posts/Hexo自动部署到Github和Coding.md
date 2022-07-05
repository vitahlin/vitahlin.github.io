---
title: Hexo自动部署到Github和Coding
tags:
  - Blog
  - Hexo
categories: Blog
comments: true
copyright: true
abbrlink: auto_deploy_hexo_to_github_and_coding
date: 2018-12-10 12:00:33
updated: 2018-12-10 12:00:33
---

### 概述

Hexo 是一个快速、简洁且高效的博客框架。
本文介绍如何使用`docker`管理`hexo`环境以及用`Travis CI`实现自动部署到Github Page和Coding Pag上。

<!--more-->

#### 使用`docker`

使用`docker`可以不需要每次重新安装`hexo`环境。

我们用`docker-compose`实现`hexo`的镜像，这样本地只需要`docker`即可，具体配置及其使用方法可以参考这里：[https://github.com/vitahlin/Dockers/tree/master/hexo](https://github.com/vitahlin/Dockers/tree/master/hexo)

#### `hexo`使用

在容器内执行对应的`hexo`命令即可。

#### 使用`Travis CI`实现自动部署

1. 关联`Travis CI`和`Github`账号

这个直接用`Github`账号登录即可。

2. 设置要执行`Travis CI的项目` 
在链接[https://www.travis-ci.org/account/repositories](https://www.travis-ci.org/account/repositories)中，同步`Github`账号中的代码库，勾选即可。
勾选后，要在当前项目的代码根目录下创建一个`.travis.yml`文件，用来配置要执行的内容。

如图所示：
![auto_deploy_hexo_to_github_and_coding_1](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/auto_deploy_hexo_to_github_and_coding_1.png)
打勾的即是要执行`Travis CI`的项目，点击`Setting`可以进行具体的设置。

3. 环境变量配置

点击`Setting`进行具体的设置，我们可以在`Environment Variables`中配置需要的环境变量。

##### Github Page token设置

为了能够正确提交到`Github Page`，有两种方式，一种是通过`token`来执行操作，一种是通过`SSH Keys`，这里介绍使用`token`的方式。

在[https://github.com/settings/tokens](https://github.com/settings/tokens)页面点击`Generate new token`申请`token`，需要注意的是，需要给`token`赋予足够的权限，如图：
![auto_deploy_hexo_to_github_and_coding_2](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/auto_deploy_hexo_to_github_and_coding_2.png)
点击生成后，可以在环境变量中配置该`token`。

生成`token`后，就可以通过该`token`来操作，命令如下：
```c 
git clone https://user_tokenM@github.com/user_name/user_repo_name.git master
```

- user_token 对应生成的`token`
- user_name 对应你的用户名
- user_repo_name 对应仓库名
- master 即分支名，可以改成自己的分支

##### Coding token配置
`Coding`类似，在该页面新建`token`：[https://dev.tencent.com/user/account/setting/tokens/new](https://dev.tencent.com/user/account/setting/tokens/new)

`token`的权限需要选择**project:depot  读/写 完整的仓库控制权限** 这一项。

Coding的`token`是可以更新权限的，更新的话好像有时候并不及时，可以直接删掉新建`token`。

Coding如何通过`token`访问个人仓库可以参考：[https://open.coding.net/references/personal-access-token/#利用令牌访问代码仓库](https://open.coding.net/references/personal-access-token/#%E5%88%A9%E7%94%A8%E4%BB%A4%E7%89%8C%E8%AE%BF%E9%97%AE%E4%BB%A3%E7%A0%81%E4%BB%93%E5%BA%93)

##### 环境变量配置

生成`token`之后，我们就可以通过`token`来将代码推送到GitHub Page对应的分支上面。
把推送需要的变量配置在`Environment Variables`中，如下图所示：
![auto_deploy_hexo_to_github_and_coding_3](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/auto_deploy_hexo_to_github_and_coding_3.png)

当配置比较私密的信息时候，如`token`则不要勾选`Display value in build log`，否则会将该值显示在`log`内容中。
环境变量配置完，则配置`.travis.yml`来执行。

4. `.travis.yml`文件
具体内容可以参考：[https://github.com/vitahlin/vitahlin.github.io/blob/develop/.travis.yml](https://github.com/vitahlin/vitahlin.github.io/blob/develop/.travis.yml)

- branches `only`设置指定分支生效，只有当该项目提交到该分支的时候才执行

当前源码在：[https://github.com/vitahlin/vitahlin.github.io/tree/develop](https://github.com/vitahlin/vitahlin.github.io/tree/develop)

即`vitahlin.github.io`的`develop`分支，当有代码提交到该分支时，就会执行`Travis CI`构建，将`Hexo`自动部署到当前的`master`分支和Coding Page的`master`分支。

部署过程中如果失败，会有发送提示邮件到对应邮箱，具体日志可以在[https://www.travis-ci.org/](https://www.travis-ci.org/)中查看。
