---
title: "Homebrew安装指定版本的软件"
date: 2020-05-06T16:29:02Z
draft: false
lightgallery: true
---

# Homebrew安装指定版本的软件

下面以`gradle`为例子，首先查看`gradle`的最新版本：

```c
$ brew info gradle

gradle: stable 6.0.1
Open-source build automation tool based on the Groovy and Kotlin DSL https://www.gradle.org/ 
Not installed
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/gradle.rb
==> Requirements
Required: java >= 1.8 ✔
==> Analytics
install: 29,004 (30 days), 145,995 (90 days), 612,138 (365 days)
install-on-request: 28,126 (30 days), 138,808 (90 days), 578,499 (365 days)
build-error: 0 (30 days)
```

可以看到，最新的`gradle`版本是`6.0.1`，这时候假设我们想安装`5.5.1`版本的`gradle`。

克隆`Homebrew-code`，查看`gradle`指定版本对应的提交记录：

```bash
$ git clone git@github.com:Homebrew/homebrew-core.git
$ cd homebrew-core
$ git log master -- Formula/gradle.rb

……
commit 5b33b4ee81efc4c0d6187bd32b2d3c1401bfdc9e
Author: Sterling Greene <big-guy@users.noreply.github.com>
Date:   Wed Aug 14 18:04:25 2019 -0400

    gradle 5.6

    Closes #43119.

    Signed-off-by: Chongyu Zhu <i@lembacon.com>

commit 70cfae2f21908d936f2cb87e71bf4c68b4e89ea6
Author: Sterling Greene <big-guy@users.noreply.github.com>
Date:   Wed Jul 10 21:48:44 2019 -0400

    gradle 5.5.1 (#41828)

commit 783b91c0aa72719e580bbfde633cbfa86d8cda40
Author: Sterling Greene <big-guy@users.noreply.github.com>
Date:   Fri Jun 28 17:12:23 2019 -0400

    gradle 5.5 (#41432)
……
```

可以看到，`5.5.1`版本对应的`commit hash`是：`70cfae2f21908d936f2cb87e71bf4c68b4e89ea6`，然后直接通过下面命令直接安装`5.5.1`版本的`gradle`：

```bash
brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/70cfae2f21908d936f2cb87e71bf4c68b4e89ea6/Formula/gradle.rb
```

只要我们能找到对应的COMMIT-HASH，即可安装任意版本的软件：

```c
brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/<COMMIT-HASH>/Formula/<Software>.rb

# Cask
brew cask install https://raw.githubusercontent.com/Homebrew/homebrew-cask/<COMMIT-HASH>/Casks/<SOFTWARE>.rb
```

如果近期不打算更新`gradle`，可以用`pin`命令把`gradle`“钉“在当前版本，防止不下心运行`brew upgrade`命令导致的更新：

```bash
$ brew pin gradle
```

通过这个命令，还可以列出哪些软件是被`pin`的：

```bash
$ brew list --pinned
```
