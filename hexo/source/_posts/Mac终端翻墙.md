---
title: Mac终端翻墙
tags: Linux&Unix
categories: Linux&Unix
comments: true
abbrlink: proxy_in_terminal
date: 2016-11-24 18:26:33
updated: 2016-11-24 18:26:33
copyright: true
---

**系统版本macOS 10.12.1，需要`shadowsocks`支持，默认系统已安装Homebrew**

### 安装`proxychains-ng`

```c
brew install proxychains-ng
```

### 设置

编辑配置文件`vim /usr/local/etc/proxychains.conf`在 [ProxyList] 下面（也就是末尾）加入代理类型，代理地址和端口，例如使用 TOR 代理，注释掉原来的代理并添加`socks5 127.0.0.1 1080`

<!--more-->

```c
// 打开配置文件
vim /usr/local/etc/proxychains.conf
// 修改ProxyList配置
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
#socks4 	127.0.0.1 9050
socks5 127.0.0.1 1080
```

### 使用

安装时在命令前加上`proxychains4`，如：

```c
proxychains4 brew install XXX
```

如果使用`zsh`可设置`.zshrc`文件简化命令：

```c
vi ~/.zshrc
alias pc=proxychains4
```

保存后运行`source ~/.zshrc`。
这样的话命令`proxychains4 brew install XXX`可简化为`pc brew install XXX`

#### 测试

```c
proxychains4 curl google.com
```

#### 使用`proxychains4`时提示警告

```c
dyld: warning: could not load inserted library '/usr/local/Cellar/proxychains-ng/4.11/lib/libproxychains4.dylib' into library validated process because no suitable image found.  Did find:
	/usr/local/Cellar/proxychains-ng/4.11/lib/libproxychains4.dylib: code signing blocked mmap() of '/usr/local/Cellar/proxychains-ng/4.11/lib/libproxychains4.dylib'
```

使用`proxychains4`配合brew安装时可能会提示如上警告，好像直接忽略就可以。有时候停住不动用`Ctrl+c`继续。



#### 参考
- [https://segmentfault.com/a/1190000004607285](https://segmentfault.com/a/1190000004607285)
- [http://apple.stackexchange.com/questions/253401/proxychains-suddenly-stopped-working](http://apple.stackexchange.com/questions/253401/proxychains-suddenly-stopped-working)
