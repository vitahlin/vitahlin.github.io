---
title: ssh证书登录
<!-- description: 这是一个副标题 -->
date: 2022-07-20
slug: use-ssh-key-to-login
categories:
    - tools

tags:
    - tools
---

# 证书登陆的步骤

1. 客户端生成证书:私钥和公钥，然后私钥放在客户端，妥当保存，一般为了安全，访问有黑客拷贝客户端的私钥，客户端在生成私钥时，会设置一个密码，以后每次登录ssh服务器时，客户端都要输入密码解开私钥
2. 服务器添加信用公钥：把客户端生成的公钥，上传到ssh服务器，添加到指定的文件中，这样，就完成ssh证书登录的配置了。

假设客户端想通过私钥要登录其他ssh服务器，同理，可以把公钥上传到其他ssh服务器。

真实的工作中:员工生成好私钥和公钥(千万要记得设置私钥密码)，然后把公钥发给运维人员，运维人员会登记你的公钥，为你开通一台或者多台服务器的权限，然后员工就可以通过一个私钥，登录他有权限的服务器做系统维护等工作，所以，员工是有责任保护他的私钥的。


# 客户端建立私钥和公钥

```c
ssh-keygen -t rsa
```
会在`.ssh`目录下生成两个文件，分别是私钥 (id_rsa) 与公钥 (id_rsa.pub)。另外就是私钥的密码了，如果不是测试，不是要求无密码ssh，不能输入空(直接回车)，要妥当想一个有特殊字符的密码。

# `SSH`服务端配置

```c
vim /etc/ssh/sshd_config
#禁用root账户登录，非必要，但为了安全性，请配置
PermitRootLogin no

# 是否让 sshd 去检查用户家目录或相关档案的权限数据，
# 这是为了担心使用者将某些重要档案的权限设错，可能会导致一些问题所致。
# 例如使用者的 ~.ssh/ 权限设错时，某些特殊情况下会不许用户登入
StrictModes no

# 是否允许用户自行使用成对的密钥系统进行登入行为，仅针对 version 2。
# 至于自制的公钥数据就放置于用户家目录下的 .ssh/authorized_keys 内RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile      %h/.ssh/authorized_keys

#有了证书登录了，就禁用密码登录吧，安全要紧
PasswordAuthentication no
```

**要注意的是，禁止密码登录要等ssh证书全部配置好以后再设置，不然证书没配置好、密码也不能登录到时候就悲剧了。**

# 客户端公钥上传到服务器

在客户端执行命令：

```c
scp ~/.ssh/id_rsa.pub user@<ssh_server_ip>:~
```

在服务端执行命令：

```c
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

如果提示文件不存在，那么要先创建`authorized_keys `文件（这里以root用户为例）：

```c
mkdir .ssh
touch .ssh/authorized_keys
```


# 重启`SSH`

Centos7重启ssh命令：

```c
systemctl restart sshd.service
```

Centos6.4重启ssh命令：
```c 
service sshd restart
```

# 客户端通过私钥登录服务端

```c
ssh -i /user/.ssh/id_rsa user@<ssh_server_ip>
```


# 参考链接
- [http://www.cnblogs.com/ggjucheng/archive/2012/08/19/2646346.html](http://www.cnblogs.com/ggjucheng/archive/2012/08/19/2646346.html)



