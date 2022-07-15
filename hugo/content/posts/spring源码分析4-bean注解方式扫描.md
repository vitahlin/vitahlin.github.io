---
title: spring源码分析4-bean扫描
<!-- description: 这是一个副标题 -->
date: 2022-07-14
slug: use-AnnotationConfigApplicationContext-to-scan-bean
categories:
    - spring

tags:
    - spring
    - java

draft: true
---

早期的Java版本并没有支持注解，所以当时的Spring选择了XML这种语言来描述Bean，后续Java在1.5发布了注解后，Spring在3.0开始大批量引入Annotation。而基于注解注册和组件扫描的容器上下文即  `AnnotationConfigApplicationContext`。

# AnnotationConfigApplicationContext使用示例

```java
package org.springframework.vitahlin;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = "org.springframework.vitahlin.bean")
public class BeanScanConfig {
}
```

```java
package org.springframework.vitahlin.bean;

import org.springframework.stereotype.Component;

@Component
public class HelloService {
    public void helloSpring(){
        System.out.println("Hello Spring!");
    }
}
```

```java
package org.springframework.vitahlin;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.vitahlin.bean.HelloService;

public class AnnotationConfigApplicationContextTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);
        HelloService helloService = (HelloService) context.getBean("helloService");
        System.out.println(helloService);
        helloService.helloSpring();
    }
}
```

运行结果：

```shell

```