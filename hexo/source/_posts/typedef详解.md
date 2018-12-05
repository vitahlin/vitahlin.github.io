---
title: typedef详解
tags: C++
categories: C++
comments: true
copyright: true
abbrlink: 1ddcff8f
date: 2016-11-29 20:19:33
updated: 2016-11-29 20:19:33
---

概述：是一种在计算机编程语言中用来声明自定义数据类型，配合各种原有数据类型来达到简化编程的目的的类型定义关键字。
 
用法:（重点在于与结构结合使用）

```c 
typedef int elemtype;
```
    
此时`elemtype`等价于`int`,即可用`elemtype a;`语句定义一个`int`类型的数据。

<!--more-->

与结构结合使用,例如：

```c
typedef struct tagMyStruct
{ 
    int iNum;
    long lLength;
} MyStruct;
```

这语句实际上完成两个操作：

-  定义一个新的结构类型

```c
struct tagMyStruct
{ 
    int iNum; 
    long lLength; 
};
```

分析：`tagMyStruct`称为“tag”，即“标签”，实际上是一个临时名字，`struct`  关键字和`tagMyStruct`一起，构成了这个结构类型，不论是否有`typedef`，这个结构都存在。**我们可以用`struct tagMyStruct varName`来定义变量，但要注意，使用`tagMyStruct varName`来定义变量是不对的，因为`struct`和`tagMyStruct`合在一起才能表示一个结构类型.**

 -  ` typedef`为这个新的结构起了一个名字，叫`MyStruct`。
 
```c
typedef struct tagMyStruct MyStruct;
``` 

因此，`MyStruct`实际上相当于`struct tagMyStruct`，我们可以使用`MyStruct varName`来定义变量。
