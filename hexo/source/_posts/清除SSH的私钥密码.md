---
title: 清除SSH的私钥密码
tags: Linux&Unix
categories: Linux&Unix
comments: true
copyright: true
abbrlink: 5a76c088
date: 2017-07-18 22:20:33
updated: 2017-07-18 22:20:33
---

我们为了保证 SSH 私钥的安全可以在生成的时候为之设置密码，但是日常使用中要一直输入太过麻烦，可以用如下方法取消掉。

在终端中输入 `ssh-keygen -p` 即可：

```c
Enter file in which the key is (/Users/vitah/.ssh/id_rsa):
Enter old passphrase:
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
```

可以看到会提示选择要更改密码的 SSH 私钥文件，然后输入旧密码，提示输入新密码时直接 Enter 即可免密。

同理，我们也可以用这个命名来更改或者增加密码。