---
title: maxOS环境下终端配置
<!-- description: 这是一个副标题 -->
date: 2022-07-04
slug: mac-terminal
categories:
    - tools

tags:
    - tools
    
---

终端软件很多，比如很流行的终端软件[iTerm2](https://iterm2.com)，但这里不做介绍，macOS其实原生自带了终端工具 Terminal，我们来了解一下如何美化。

## 字体安装

大部分主题都需要特殊的字体支持，所以一开始我们需要下载这些 `powerline` 字体。

字体下载：[https://github.com/powerline/fonts](https://github.com/powerline/fonts)

可以按如下命令下载并导入：

```shell
# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```

## Terminal主题美化

下载主题美化Terminal，下载地址：[https://github.com/mbadolato/iTerm2-Color-Schemes](https://github.com/mbadolato/iTerm2-Color-Schemes)

下载到本地后，打开Terminal，导入所选主题即可，如图所示：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207051436559.png)

这里导入主题 OneHalfDark，效果如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207051437188.png)

## zsh配置

主题美化后，我们需要安装一下插件来优化 zsh 操作。
首先安装 zinit，用它来管理 zsh 插件，关于 zinit 的介绍网上很多，主要特点是快并且易于配置，一个 .zshrc文件即可拷贝全部配置。参考链接：

- [Zsh & Zinit 配置舒服的终端环境](https://kissandrun.site/shell-3/)
- [使用 zinit 管理 zsh 插件 完美代替 Antigen](https://einverne.github.io/post/2020/10/use-zinit-to-manage-zsh-plugins.html)
- [加速你的 zsh —— 最强 zsh 插件管理器 zplugin/zinit 教程](https://www.aloxaf.com/2019/11/zplugin_tutorial/)

通用配置：

```shell
# 科学上网需要
alias fq="export ALL_PROXY=127.0.0.1:9999"
alias nfq="unset ALL_PROXY"
alias ip="curl ipinfo.io"

alias vi="mvim"
alias zshrc="vi ~/.zshrc"
alias vimrc="vi ~/.vimrc"
alias rm="trash
```

安装完成后，我们还需要安装插件来优化 zsh。

### 主题优化

#### zsh主题 powerlevel10k

zshrc 代码：

```shell
# p10k主题
zinit ice depth"1" 
zinit light romkatv/powerlevel10k
```

安装完成后，终端输入 `p10k configure`来自定义p10k主题配置。

参考链接：[https://github.com/romkatv/powerlevel10k](https://github.com/romkatv/powerlevel10k)

#### 语法高亮

```shell
# 语法高亮
zinit ice lucid wait='0' atinit='zpcompinit'
zinit light zdharma/fast-syntax-highlighting
```

参考链接：[https://github.com/zdharma-continuum/fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting)

如果需要 `ls` 命令也能高亮颜色的话，还需要如下配置：

```shell
# ls和cd命令展示文件夹颜色
export CLICOLOR=1
export LSCOLORS=ExGxFxdaCxDaDahbadeche
zstyle ':completion:*' list-colors "${(@s.:.)LS_COLORS}"
```

### 操作优化

#### 目录快速跳转 zsh-z

参考链接：[https://github.com/agkozak/zsh-z](https://github.com/agkozak/zsh-z)

```shell
# 快速跳转，需要安装lua，brew install lua
zinit ice lucid wait='1'
zinit light skywind3000/z.lua
alias zb="z -b"
```

该插件需要 `lua` 支持，可以通过 `Homebrew` 安装：`brew install lua`

#### 终端vim模式 zsh-vi-mode

参考链接：[https://github.com/jeffreytse/zsh-vi-mode](https://github.com/jeffreytse/zsh-vi-mode)

```shell
终端vim模式
zinit ice depth=1
zinit light jeffreytse/zsh-vi-mod
```
