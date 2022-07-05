---
title: Mongoose基本用法
tags: Database
categories: Database
comments: true
copyright: true
abbrlink: how_to_use_mongoose
date: 2017-07-12 14:58:33
updated: 2017-07-12 14:58:33
---

### Mongoose介绍
`Mongoose` 是 `MongoDB` 的一个对象模型工具，是基于`node-mongodb-native`开发的MongoDB 的 Node.js 驱动，可以在异步的环境下执行。同时它也是针对 `MongoDB` 操作的一个对象模型库，封装了 MongoDB 对文档的一些增删改查等常用方法，让 Node.js 操作 MongoDB 数据库变得更加容易。

<!--more-->

### Mongoose的基本概念

#### 连接数据库

##### 数据库连接配置

我们可以新建一个`json`文件来保存连接数据库的各种配置，`mongodb_config.json`:
```javascript
{
    "database": "PomeloDemo",
    "host": "127.0.0.1",
    "options": {
        "pass": "",
        "server": {
            "auto_reconnect": true,
            "socketOptions": {
                "keepAlive": 1
            }
        },
        "user": ""
    },
    "port": 27017
}
```

##### 连接数据库
然后就可以在代码中实现数据库的连接，下述代码中的`mongodb_config`就是上面的配置文件：

```javascript
var mongoose = require('mongoose');

// 使用mongoose连接mongodb
var db = mongoose.connect(mongodb_config.host, mongodb_config.database, mongodb_config.port, mongodb_config.options).connection;

// 当数据库连接失败时提示
db.on('error', console.error.bind(console, 'connection error:'));

// 连接成功执行成功回调
db.once('open', function callback() {
	console.log('数据库连接成功');
});
```

#### schema和模型

我们需要定义一个`schema`，描述此集合里面有哪些字段，字段是什么类型，如下所示，我们定义了一个用户表，包含用户的ID、等级和名字信息。新建`roleSchema.js`文件：
```javascript
var roleSchema = new mongoose.Schema({
    // 角色id
    uid: {
        type: Number,
        required: true,
        unique: true,
        index: true
    },

    // 玩家等级
    lv: {
        type: Number,
        default: 1
    },

    // 玩家昵称
    name: {
        type: String,
        default: 'Vitah'
    }
});

roleSchema.methods.getName = function () {
    return this.name;
};

roleSchema.methods.setName = function (new_name) {
    this.name = new_name;
};

// 创建Role模型
mongoose.model('Role', roleSchema);
```

#### 实体和数据保存
接下来我们就可以根据模型创建实体，例如：
```c
var Role = mongoose.model('Role');
var xiaoming = new Role({
    uid:1,
    lv:10,
    name:'小明'
});
```

然后调用`save`方法保存到数据库中：
```javascript
xiaoming.save(function(error,doc){
    if(error){
        console.log("error :" + error);
    }else{
        console.log(doc);
    }
});
```