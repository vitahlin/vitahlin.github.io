---
title: spring源码2-各模块介绍
<!-- description: 这是一个副标题 -->
date: 2022-07-07
slug: introduction-to-spring-framework-modules
categories:
    - spring

tags:
    - spring
    - java
---

Spring中的模块划分如下图所示，除了图中的spring-vitahlin外，还有24个模块：
![spring模块](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207071923185.png)

## 1. framework-bom

通过该模块，可以解决Spring中的模块与其他框架整合时产生jar包版本的冲突，默认为空实现。

## 2. integration-tests

spring的集成测试模块。

## 3. spring-aop

面向切面编程时使用。Spring通过"横切"的方式将贯穿于多业务中的公共功能独立抽取出来，形成单独的切面，并指定切面的具体动作，在需要使用该功能时，动态地将该功能切入到需要的地方。

## 4. spring-aspects

用来实现 `AspectJ` 框架的集成。`AspectJ` 是一个通过Java扩展出之后的框架，框架里面定义了 AOP 的语法，通过特殊的编译器完成编译期间的代码植入，最后生成增强之后的 Class 文件。

## 5. spring-beans

完成spring框架的基本功能，里面定义了大量和Bean有关的接口、类和注解。例如，bean定义的顶层接口 `BeanDefinition`，Bean装配相关的注解 `Autowired`、`Qualifier`、`Value`，用来创建bean的工厂接口 `BeanFactory` 和一些具体的工厂方法等。

## 6. spring-context

用来实现spring的上下文功能，以及spring的IOC，例如初始化spring容器时所使用的 `ApplicationContext` 接口和常用的抽象实现类 `AnnotationConfigApplicationContext` 或者 `ClasspathXmlApplicationContext` 等。

## 7. spring-context-indexer

用来创建spring应用启动时候所选组件的索引，以提高应用的启动速度。通常情况下，应用启动的时候会扫描类路径下所有组件，但是如果组件特别多，会导致应用启动特别慢。该模块可以在应用的编译器对应用的类路径下的组件创建索引，在启动的时候通过索引去加载和初始化组件，可以大大提高应用启动速度。

## 8. spring-context-support

用来提供spring上下文的一些扩展模块，例如实现邮件服务、视图解析、缓存的支持、定时任务调度等。

## 9. spring-core

spring的核心功能实现，例如控制反转(IOC)、依赖注入(DI)、`asm` 以及 `cglib` 的实现。

## 10. spring-expression

提供spring表达式语言等支持，SPEL。

## 11. spring-instrument

实现spirng对服务器的代理接口功能实现，实现的是类级别或者 `ClassLoader` 级别的代理功能。

## 12. spring-jcl

通过适配器设计模式实现的一个用来统一管理日志的框架，对外提供统一的接口，采用“适配器”将日志的操作全部委托给具体的日志框架，提供了对多种日志框架的支持。

## 13. spring-jdbc

spring对JDBC(Java Data Base Connector)功能的支持，里面定义了用于操作数据的多种API，常用的即 JdbcTemplate。通过模版设计模式将数据库的操作和具体业务分离，降低了数据库操作和业务功能的耦合。

## 14. spring-jms

对Java消息服务的支持，对JDK中的JMS API进行了简单的封装。

## 15. spring-messaging

实现基于消息来构建服务的功能。

## 16. spring-orm

提供一些整合第三方ORM框架的抽象接口，用来支持与第三方ORM框架进行整合，例如：MyBatis、Hibernate、Spring JPA等。

## 17. spring-oxm

spring用来对对象和 xml(Object/xml) 映射的支持，完成 xml 和 object 对象的相互转换。

## 18. spring-r2dbc

r2dbc 是支持使用反应式编程API访问关系型数据库的桥梁，定义统一接口规范，不同数据库厂家通过实现该规范提供驱动程序包。

## 19. spring-test

Spring对Junit测试框架的简单封装，用来快速构建应用的单元测试功能及Mock测试。

## 20. spring-tx

对一些数据访问框架提供的声明式事务或者编程式事务（通过配置文件进行事务的声明）的支持。例如 Hibernate、Mybatis、JPA 等。

## 21. spring-web

用来支持 web 系统的功能。例如文件上传、与JSF的集成、规律器 Filter 的支持等。

## 22. spring-webflux

spring5中新增的一个通过响应式编程来实现 web 功能的框架。内部支持了 reactive 和非阻塞式的功能。例如可以通过tcp的长连接来实现数据传输。webmvc的升级版，webmvc是基于 servlet 的，而 webflux 是基于 reactive 的。

## 23. spring-webmvc

用来支持SpringMVC的功能，包括了和SpringMVC框架相关的所有类或者接口，例如常用的DispatcherServlet、ModelAndView、HandlerAdaptor等。另外提供了支持国际化、标签、主题、FreeMarker、Velocity、XSLT的相关类。注意：如果使用了其他类似于smart-framework的独立MVC框架，则不需要使用该模块中的任何类。

## 24. spring-websocket

Spring对websocket的简单封装，提供了及时通信的功能，常用于一些即时通讯功能的开发，例如：聊天室。
