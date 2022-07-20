---
title: git忽略设置
<!-- description: 这是一个副标题 -->
date: 2022-07-05
slug: gitignore-settings
categories:
    - other

tags:
    - other
---

在项目目录下新建`.gitignore`文件，`.gitignore`文件对其所在的目录及所在目录的全部子目录均有效。通过将.gitignore文件添加到仓库，其他开发者更新该文件到本地仓库，可以共享同一套忽略规则。

## 基本的配置规则

- 忽略＊.o和＊.a文件
   - ＊.[oa]
- 忽略＊.b和＊.B文件,my.b除外:
   - ＊.[bB]
   - !my.b
- 忽略dbg文件和dbg目录
   - dbg
- 只忽略dbg目录，不忽略dbg文件
   - dbg/
- 只忽略dbg文件，不忽略dbg目录
   - dbg
   - !dbg/
- 只忽略当前目录下的dbg文件和目录，子目录的dbg不在忽略范围内 
   - /dbg

## 从Github上下载忽略文件

Github上面有个项目专门收集各种忽略文件，我们可以直接从中下载对应的忽略文件到本地项目中进行应用，地址如下：[gitignore](https://github.com/github/gitignore)

这里以`C.gitignore`文件为例，找到该文件，点击文件的`Raw`查看文件的地址，然后用如下命令下载`C.gitignore`：

```shell
wget --no-check-certificate https://raw.githubusercontent.com/github/gitignore/master/C.gitignore -O .gitignore
```

链接地址为上述中查看的地址，下载后即可。`-O .gitignore`表示把下载的文件重新命令为`.gitignore`。

## `.gitignore`忽略规则不生效解决方法

有时候在项目开发过程中，突然心血来潮想把某些目录或文件加入忽略规则，按照上述方法定义后发现并未生效，原因是 **.gitignore 只能忽略那些原来没有被track的文件**，如果某些文件已经被纳入了版本管理中，则修改 .gitignore是 无效的。那么解决方法就是**先把本地缓存删除（改变成未track状态），然后再提交**：

```c
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```

## 参考链接

- [https://segmentfault.com/q/1010000003410211](https://segmentfault.com/q/1010000003410211)
- [http://www.pfeng.org/archives/840](http://www.pfeng.org/archives/840)
