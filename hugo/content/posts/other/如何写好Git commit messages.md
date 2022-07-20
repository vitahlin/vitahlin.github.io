---
title: 如何写好git commit message
<!-- description: 这是一个副标题 -->
date: 2022-07-20
slug: how-to-write-git-commit-message
categories:
    - other

tags:
    - other
---

任何软件项目都是一个协作项目，它至少需要2个开发人员参与，当原始的开发人员将项目开发几个星期或者几个月之后，项目步入正规。不过他们或者后续的开发人员仍然需要经常提交一些代码去修复bug或者实现新的feature。我们经常有这种感受：当一个项目时间过了很久之后，我们对于项目里面的文件和函数功能渐渐淡忘，重新去阅读熟悉这部分代码是很浪费时间并且恼人的一件事。但是这也没法完全避免，我们可以使用一些技巧尽可能减少重新熟悉代码的时间。commit messages 可以满足需要，它也反映了一个开发人员是否是良好的协作者。

编写良好的Commit messages可以达到3个重要的目的：
- 加快review的流程
- 帮助我们编写良好的版本发布日志
- 让之后的维护者了解代码里出现特定变化和feature被添加的原因



这里有一个比较好的示例，可以参考一下。
![](http://oc1mf55gf.bkt.clouddn.com/git20170221001.png)


关于Commit messages 的格式有多种，不同的社区可能有各自不同的格式，本文介绍的这种做参考即可。


# Commit messages基本语法

Commit messages的基本语法
```c 
<type>: <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

`Type` 表示提交类别，`Subject` 表示标题行，`Body` 表示主体描述内容。

`Type` 的类别说明：
- feat: 添加新特性(feature)
- fix: 修复bug
- docs: 仅仅修改了文档
- style: 仅仅修改了空格、格式缩进等等，不改变代码逻辑
- refactor: 代码重构，即不是新增功能，也不是修改bug的代码变动
- perf: 增加代码进行性能测试
- test: 增加测试用例
- chore: 改变构建流程、或者增加依赖库、工具等

# Commit messages格式要求

```c 
# 标题行：50个字符以内，描述主要变更内容
#
# 主体内容：更详细的说明文本，建议72个字符以内。 需要描述的信息包括:
#
# * 为什么这个变更是必须的? 它可能是用来修复一个bug，增加一个feature，提升性能、可靠性、稳定性等等
# * 他如何解决这个问题? 具体描述解决问题的步骤
# * 是否存在副作用、风险? 
#
# 如果需要的话可以添加一个链接到issue地址或者其它文档
```

# Commit messages书写建议
- 尽可能多的提交，单个Commit messages包含的内容不要过多
- 标题行以Fix, Add, Change, Delete开始，采用命令式的模式。不要使用Fixed, Added, Changed等等
- 始终让第二行是空白，不写任何内容
- 主体内容注意换行，避免在git里面滚动条水平滑动
- 永远不在 git commit 上增加 -m 或 --message= 参数，提交的时候git commit即可。

# 参考链接
- Git 写出好的 commit message [https://ruby-china.org/topics/15737](https://ruby-china.org/topics/15737)
- Git 的正确提交姿势 [https://www.oschina.net/news/69705/git-commit-message-and-changelog-guide](https://www.oschina.net/news/69705/git-commit-message-and-changelog-guide)