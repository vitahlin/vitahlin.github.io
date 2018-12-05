---
title: 阿里云部署Ghost
tags: Blog
categories: Blog
comments: true
abbrlink: f331b204
date: 2016-07-11 00:04:33
updated: 2016-07-11 00:04:33
copyright: true
---

系统环境是CentOS 7 64位。

## 安装Nginx

###  系统更新

#### 安装内置的EPEL源

```c
yum install epel-release
rpm -Uvh https://rhel7.iuscommunity.org/ius-release.rpm
rpm -Uvh https://centos7.iuscommunity.org/ius-release.rpm
```
安装中若是出现失败的情况只要多运行几次即可。

#### 更新yum缓存

```c
yum repolist
```

<!--more-->

#### 更新CentOS7内核和内置软件

```c
yum update -y
```

更新需要一段时间，大概十分钟左右，完成后重启服务器，如果有其他项目正在运行，重启服务器要谨慎。同样更新中若是出现网络不可达失败等情况，多试几次即可。



### 安装

#### 配置Nginx yum源

新建并编辑Nginx yum源文件：

```c
vim /etc/yum.repos.d/nginx.repo
```

进入vim编辑模式，在该文件中输入以下内容:

```c
[nginx] 
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/ 
gpgcheck=0
enabled=1
```

然后保存。



#### 安装Nginx

```c
yum install nginx -y // 安装Nginx
systemctl enable nginx // 设置Nginx随服务器启动而启动
systemctl start nginx // 启动Nginx
systemctl status nginx // 查看Nginx状态
```

看到如下内容则说明启动成功：

```c
Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
Active: active (running) since 一 2016-05-30 15:31:57 CST; 6s ago
```
这时候在浏览器中输入阿里云的公网地址，可以看到Welcome o nginx!


#### 设置Nginx反向代理

如果完成上一步并且在浏览器中看见Nginx的welcome，那么我们就可以开始配置反向代理，让代理从80端口指向2368端口，去到以下这个目录：
```c
cd /etc/nginx/conf.d
```
里面有一个default.conf的配置文件，删掉这个文件或者把这个文件里面的内容全部用#注释掉，填写新的内容:
```c
server {  
    listen 80;
    server_name zyden.vicp.cc; // 自己的域名或者ip地址
    location / {
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   Host      $http_host;
        proxy_pass         http://127.0.0.1:2368;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

保存退出，并重启nginx：

```c
service nginx restart
```

<br/>

---

## 安装Mysql


### 错误的安装方法

使用常规的命令：

```c
yum -y install mysql mysql-server mysql-devel
```

安装时不会安装Mysql数据库，而是安装离mariadb数据。因为在CentOS 7和CentOS 7.1系统中，默认安装的mysql是它的分支mariadb。这里引用下百度百科关于mariadb的描述：

>MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可 MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。开发这个分支的原因之一时：甲骨文公司收购了Mysql后，有将Mysql闭源的潜在风险，因此社区采用分支的方式来避开这个风险。


### 正确的安装方法

Linux系统自带的repo是不会自动更新每个软件的最新版本（基本都是比较靠后的稳定版），所以无法通过yum方式安装MySQL的高级版本。所以我们需要先安装带有当前可用的mysql5系列社区版资源的rpm包。

#### 安装rpm包：

```c
rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```



#### 查看当前可用的Mysql安装资源：

```c
[root@iZ23dnc7nkxZ ~]# yum repolist enabled | grep "mysql.*-community.*"
```

输出内容：
```c
mysql-connectors-community/x86_64 MySQL Connectors Community                  21
mysql-tools-community/x86_64      MySQL Tools Community                       33
mysql56-community/x86_64          MySQL 5.6 Community Server                 229
```

从上面列表可以看出mysql56-community/x86_64和MySQL 5.6 Community Server可以使用。

#### 直接用yum方式安装Mysql5.6版本

```c
yum -y install mysql-community-server
```

等待安装完成即可。



### 安装后的设置

#### 启动mysql：

```c
systemctl start mysqld
```

#### 设置为开机启动

```c
systemctl enable mysqld
```

#### 添加相关的mysql配置：

```c
Enter current password for root (enter for none): //提示输入当前的root密码，还未设置，直接Enter即可
Set root password? [Y/n] y // 输入y并设置root密码
Remove anonymous users? [Y/n] y //删除匿名用户
Disallow root login remotely? [Y/n] y // 禁止root远程登录
Remove test database and access to it? [Y/n] y // 删除test测试数据库
Reload privilege tables now? [Y/n] y // 刷新权限
```

#### 设置支持中文的配置

编辑Mysql的配置文件:

```c
vi /etc/my.cnf
```

在对应位置加上对应内容：

```c
[client]
default-character-set=utf8  
[mysql]
default-character-set=utf8  
[mysqld]
character-set-server=utf8
collation-server=utf8_general_ci
```

#### 新建一个ghost专用的mysql用户和专用的database：

(1) 先登录root账户：

```c
mysql -u root -p // 用root用户登录mysql
```

(2) 创建ghost数据库：

```c
create databse ghost;
```

创建之后可以通过以下命令查看数据库是否创建成功:

```c
show databases;
```

运行结果：

```c
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ghost              |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)
```

可以看到已经创建成功。



(3) 创建ghost用户：

```c
create user ghost;
```

然后通过下述命令查看是否创建成功：

```c
SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
```

运行结果：

```c
mysql> SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
+---------------------------+
| query                     |
+---------------------------+
| User: 'ghost'@'%';        |
| User: 'root'@'127.0.0.1'; |
| User: 'root'@'::1';       |
| User: 'root'@'localhost'; |
+---------------------------+
4 rows in set (0.00 sec)
```

可以看到ghost用户已经创建成功，％表示能被所有地址访问。



(4) 为用户ghost授权：

```c
mysql> grant all privileges on ghost.* to ghost@'%';
Query OK, 0 rows affected (0.00 sec)
```

通过命令查看用户ghost的权限状态：

```c
mysql> show grants for ghost@'%';
+--------------------------------------------------+
| Grants for ghost@%                               |
+--------------------------------------------------+
| GRANT USAGE ON *.* TO 'ghost'@'%'                |
| GRANT ALL PRIVILEGES ON `ghost`.* TO 'ghost'@'%' |
+--------------------------------------------------+
2 rows in set (0.00 sec)

```

这时候我们还需要为ghost用户设置密码，在shell环境下，输入以下命令：

```c
root@iZ23dnc7nkxZ ~]# mysqladmin -u ghost password "xxxxxxxx"
```

提示 `arning: Using a password on the command line interface can be insecure.`即设置成功。设置用户密码有多种方法，可以自行搜索。



<br/>

## 安装Node.js


有多种方法，下面介绍源码安装方案：

### 下载源码：

```c
wget http://nodejs.org/dist/v0.10.40/node-v0.10.40.tar.gz
```

### 解压源码：

```c
tar xzvf node-v* && cd node-v*
```

### 安装必要的编译软件

```c
yum install gcc gcc-c++
```

### 编译

```c
./configure
make
```

### 编译&安装

```c
make install
```

### 查看版本（测试安装是否成功）

```c
node --version
```

<br/>

## 安装Ghost


### 下载中文版的Ghost

将ghost安装在/var/www目录下，没有该目录则创建:

```c
// 创建该目录
cd /var
mkdir www
cd /var/www

// 下载中文版ghost
wget http://dl.ghostchina.com/Ghost-0.7.4-zh-full.zip

// 解压为ghost文件夹
unzip Ghost-0.7.4-zh-full.zip -d ghost

cd ghost
```

### 修改生产环境的配置

这里要将config.example.js重命名为config.js再对其进行修改配置：

```c
mv config.example.js config.js 
vi config.js 
```

我们找到生产环境的配置：production
>Ghost-0.7.4-zh-full这个版本默认集成 sqlite3 原生库，但博客篇幅比较大时，sqlite读写数据量太大时将会影响页面加载速度，我们可以根据个人需求改用mysql 
>如果选择使用sqlite则在config.js中只需要修改url地址
修改production中的配置，把mysql中的配置注释去掉并修改，注释sqlite3中的配置:

```c
production: {  
    url: 'XXX.XXX.XXX.XXX', //这里是你自己主机的域名，或者IP
    mail: {},
    database: {
        client: 'mysql'这里我选择使用mysql作为我博客的数据库
        connection: {
            host     : '127.0.0.1',
            user     : 'ghost', //mysql用户名
            password : 'XX', //密码
            database : 'ghost', //之前创建的ghost数据库名称
            charset  : 'utf8'
        },
    server: {
            host: '127.0.0.1',
            port: '2368'//若修改该端口记得在nginx中做相应改变
        }
    }  
```

修改后用命令`npm start -production`启动开发者模式的Ghost，然后在浏览器中输入ip地址或者域名，如果能看见Ghost则说明配置成功。



### 守护Ghost进程

只要我们一断开ssh，Ghost的进程就会被关闭，这里我们使用pm2来守护Ghost服务进程，并让其运行在生产模式production上:

```c
npm install pm2 -g
cd /var/wwwroot/ghost
NODE_ENV=production pm2 start index.js --name "ghost" // //让ghost以production模式运作，指定程序的入口index.js，并且此进程命名为ghost
pm2 startup centos // 开机启动 
pm2 save  
```

运行上面的命名，这样Ghost就可以运行在后台了。pm2要安装在全局，不然会提示问题。

pm2相关命令：

```c
pm2 restart 进程名
```