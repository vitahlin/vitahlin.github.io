---
title: __defineGetter__介绍
tags: JavaScript
categories: JavaScript
comments: true
copyright: true
abbrlink: how_to_use_defineGetter
date: 2017-08-03 20:29:33
updated: 2017-08-03 20:29:33
---

### 概述
`__defineGetter__` 方法可以将一个函数绑定在当前对象的指定属性上，当那个属性的值被读取时，你所绑定的函数就会被调用。

### 语法
```javascript
obj.__defineGetter__(prop, func)
```

- prop 一个字符串，表示指定的属性名
- func 一个函数，当 prop 属性的值被读取时自动被调用。

<!--more-->

### 示例代码
```javascript
// 请注意，该方法是非标准的：
var a = {};
a.__defineGetter__('gimmeFive', function() {
	return 5;
});
console.log(a.gimmeFive);



// 请尽可能使用下面的两种推荐方式来代替：

// 1. 在对象字面量中使用 get 语法
var b = {get gimmeFive() {
		return 5;
	}
};
console.log(b.gimmeFive); // 5

// 2. 使用 Object.defineProperty 方法
var c = {};
Object.defineProperty(c, 'gimmeFive', {
	get: function() {
		return 5;
	}
});
console.log(c.gimmeFive); // 5
```

### 引申

问题来源：[https://segmentfault.com/a/1190000002407109](https://segmentfault.com/a/1190000002407109)

*注：转载请注明该链接*
##### 问题
```javascript
(function(){
    var u = { a: 1, b: 2 };
    var r = {
        m: function(k){
            return u[k];
        }
    }
    window.r = r;
})()
var R = window.r;
alert(r.m('a'))
```

`alert`的结果是1，问题是能不能通过 `r.m` 获取 `u`？即当 `u` 不知道的时候，如何通过 `u[attribute]` 这种方式来获得 `u` 的自身。那么问题就来了，你需要传递一个`attribute`，`r.m(attribute)` 返回 `u`。

##### 解决
`u`是一个实例，给`u`绑定一个参数，当此参数调用当时候返回`u`自身就可以了。
`u`是一个`Object`的实例，它继承自`Object`，那么就给`Object.prototype`定义一个属性，使得该属性访问时调用的函数返回 `this` 就可以了，所以，解决方案如下：
```javascript
Object.prototype.__defineGetter__('uuu', function(){ return this; });
alert(R.m('uuu'));
```

此题目的精髓其实就三点：
1. 能否想到通过属性来访问自身
2. 能否想到使用原型继承来定义访问自身的属性
3. 能否知道`Object.prototype.__defineGetter__`