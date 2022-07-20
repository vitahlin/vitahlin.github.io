---
title: spring源码分析5-bean生命周期源码分析
<!-- description: 这是一个副标题 -->
date: 2022-07-14
slug: spring-bean-life-cycle
categories:
    - spring

tags:
    - spring
    - java

draft: true
---

Spring最重要的功能就是帮助程序员创建对象(也就是IOC)，而启动Spring就是为创建Bean对象 做准备，所以我们先明白Spring到底是怎么去创建Bean的，也就是先弄明白Bean的生命周期。

Bean的生命周期就是指：在Spring中，一个bean是如何生成的，如何销毁的。
