---
title: spring源码分析4-bean生命周期源码分析2
<!-- description: 这是一个副标题 -->
date: 2022-07-14
slug: spring-bean-life-cycle-2
categories:
    - spring

tags:
    - spring
    - java
draft: true
---

了解bean的扫描和合并之后，我们就来分析一下bean在spring中是怎么创建的。

直接来看bean创建的源码：
```java
@Override  
public Object getBean(String name) throws BeansException {  
   return doGetBean(name, null, null, false);  
}
```