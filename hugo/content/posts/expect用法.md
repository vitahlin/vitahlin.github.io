+++ 
date = "2021-03-16"
title = "Shell expect的介绍"
slug = "shell expect command"
tags = ["Linux","Shell"]
categories = ["Linux"]
series = []
disableComments = false
+++

我们通过Shell可以实现简单的控制流功能，如：循环、判断等。但是对于需要交互的场合则必须通过人工来干预，有时候我们可能会需要实现和交互程序如`telnet`服务器等进行交互的功能。而`expect`就使用来实现这种功能的工具。`expect`是一个免费的编程工具语言，用来实现自动和交互式任务进行通信，而无需人工干预。

<!--more-->

### expect用法 

**expect的核心是spawn expect send set**

1. **`#!/usr/bin/expect`**
这行代码需要在脚本的第一行，告诉操作系统脚本里的代码使用那一个`shell`来执行。这里的`expect`其实和Linux下的bash、windows下的`cmd`是一类东西。 

2. **`set timeout 30`**
设置超时时间，计时单位是秒，`timeout -1` 为永不超时

3. **`set localserver "root@192.168.0.169"`**
设置变量值，即c中的变量的声明和定义，上述代码的意思是`root@192.168.0.169`等同于`localserver`，当使用这个变量时，用符号`$`，如`ssh $localserver`

4. **`spawn ssh root@192.168.0.169`**
调用要执行的命令。`spawn`是进入expect环境后才可以执行的expect内部命令，给ssh运行进程加个壳，用来传递交互指令。 

5. **`expect "password:"`**
等待命令提示信息的出现，也就是捕捉用户输入的提示。上述代码意思是判断上次输出结果里是否包含`password:`的字符串，如果有则立即返回，否则就等待一段时间后返回，这里等待时长就是前面设置的超时时间.

6. **`send "ispass\r"`**
发送需要交互的值，替代了用户手动输入内容，命令字符串结尾别忘记加上`\r`，如果出现异常等待的状态可以核查一下。 

7. **`interact `**
保持会话连接，可以后续手动处理其它任务，请根据实际情况自行选择了。

8. **`expect eof`**
表示捕获终端输出信息终止


下面一个简单的代码示例：

```shell
#!/usr/bin/expect
set localserver "root@192.168.0.169"
set password "Linw1225"
spawn ssh $localserver
expect "*password:"
send "$password\r"
send "cd /data/Server/\r"
send "sh start.sh\r"
expect eof
```

代码实现了设置本地服务器地址，以及登录密码，然后通过ssh登录，登录成功后进入`/data/Server/`目录，执行`start.sh`脚本。上诉代码在Mac环境下运行成功。


**注：
值得注意的是不能以`sh script_name.sh`运行`expect`相关代码，保证运行权限后，可以通过`./script_name`方式运行。**
