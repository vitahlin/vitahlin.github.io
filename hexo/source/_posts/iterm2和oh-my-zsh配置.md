---
title: iterm2和oh-my-zsh配置
tags: 
categories: JavaScript
comments: true
copyright: true
abbrlink: use_iterm2_and_oh_my_zsh
date: 2018-12-11 19:03:33
updated: 2018-12-11 19:03:33
---

### 安装

```c
brew cask install iterm2
```

### 设置

#### 安装`oh-my-zsh`

```c
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

#### `oh-my-zsh`配置

##### 手动更新
终端执行命令：`upgrade_oh_my_zsh`

##### 安装所需字体

[https://github.com/powerline/fonts](https://github.com/powerline/fonts)

```c
# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```

安装完后提示所有字体均已下载到目录 */Library/Fonts* 下面
```c
Copying fonts...
Powerline fonts installed to /Users/vitah/Library/Fonts
```

##### 设置主题 `agnoster`
```c
vi .zshrc

// 修改ZSH_THEME
ZSH_THEME="agnoster"
```

##### 设置配色

主题颜色：
`Perferences—Profile—Colors—Color Presets ` 选择 `Solarized Dark`。

字体：
`Perferences—Profile—Text` 修改 `Font`  和 `Non-ASC II` 字体（以powerline结尾的字体即表示支持显示）。

##### 设置不显示用户名和电脑

默认情况下，终端会显示当前用户名和当前电脑名称。我们可以配置`.zshrc`文件来使之不显示。

在`.zshrc`文件中加入不同内容使之生效。

显示用户名不显示电脑名称：
```c
prompt_context() {
  if [[ "$USER" != "$DEFAULT_USER" || -n "$SSH_CLIENT" ]]; then
    prompt_segment black default "%(!.%{%F{yellow}%}.)$USER"
  fi
}
```

不显示用户名和电脑名称：
```c
prompt_context() {}
```

####  `oh-my-zsh`插件

##### `autojump`

安装了`autojump`之后，`zsh` 会自动记录你访问过的目录，通过 `j + 目录名` 可以直接进行目录跳转，而且目录名支持模糊匹配和自动补全，例如你访问过`hadoop-1.0.0`目录，输入`j hado` 即可正确跳转。`j –-stat` 可以看你的历史路径库。

安装
```c
brew install autojump
```

在`.zshrc`文件中增加插件配置即可使用：
```c
plugins=(git autojump)
```