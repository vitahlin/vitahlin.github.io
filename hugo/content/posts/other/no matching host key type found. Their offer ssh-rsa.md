---
title: ssh错误no matching host key type found. Their offer ssh-rsa
<!-- description: 这是一个副标题 -->
date: 2022-10-27
slug: solve-no-matching-host-key-their-offer-ssh-rsa
categories:
    - other

tags:
    - other
---


## 问题

macOS升级后，发现连接某些之前连接得好好的服务器突然无法连接，提示如下错误：
> Unable to negotiate with x.x.x.x port 2222: no matching host key type found. Their offer: ssh-rsa

## 解决办法

```shell
> ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa  user@myhost -p 2222
```
当然, 每次连接敲这么一长串也不太友好。

编辑用户 ssh 配置 `~/.ssh/config`，对于无法成功连接的host，增加配置项:
```ini
  HostKeyAlgorithms         +ssh-rsa
  PubkeyAcceptedKeyTypes    +ssh-rsa
```

完整的配置可能看起来像这样:
```ini
Host myhost
  Hostname 	1.1.1.1
  User user001
  IdentityFile     ~/.ssh/id_rsa
  # fixup for openssh 8.8
  HostKeyAlgorithms +ssh-rsa
  PubkeyAcceptedKeyTypes +ssh-rsa
```

或者，像我一样的懒人:
```ini
Host *
	ServerAliveInterval 10
	HostKeyAlgorithms +ssh-rsa
	PubkeyAcceptedKeyTypes +ssh-rsa
```


## 为什么会这样

根据 [Open**SSH**](https://www.openssh.com/) Release Notes

**Future deprecation notice**

> It is now possible[1] to perform **chosen-prefix attacks against the SHA-1 algorithm** for less than USD$50K.
> 
> In the SSH protocol, the “**ssh-rsa**” signature scheme uses the SHA-1 hash algorithm in conjunction with the RSA public key algorithm. OpenSSH will disable this signature scheme by default in the near future.
> 
> Note that the deactivation of “ssh-rsa” signatures does not necessarily require cessation of use for RSA keys. In the SSH protocol, keys may be capable of signing using multiple algorithms. In particular, “ssh-rsa” keys are capable of signing using “rsa-sha2-256” (RSA/SHA256), “rsa-sha2-512” (RSA/SHA512) and “ssh-rsa” (RSA/SHA1). Only the last of these is being turned off by default.

也就是说 8.8p1 版的 openssh 的 ssh 客户端默认禁用了 `ssh-rsa` 算法，但是对方服务器只支持 `ssh-rsa`，当你不能自己升级远程服务器的 openssh 版本或修改配置让它使用更安全的算法时，在本地 ssh 针对这些旧的ssh server重新启用 `ssh-rsa` 也是一种权宜之法。

## 参考链接
- https://www.openssh.com/releasenotes.html
- https://www.openssh.com/legacy.html
