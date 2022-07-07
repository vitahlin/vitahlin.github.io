---
title: spring源代码编译
<!-- description: 这是一个副标题 -->
date: 2022-07-07
slug: build-spring-framework-source-code
categories:
    - spring

tags:
    - spring
    - java
---

## 源代码下载

Github源码地址：[https://github.com/spring-projects/spring-framework/tree/v5.3.10](https://github.com/spring-projects/spring-framework/tree/v5.3.10)<br />下载后将目录 `spring-framework-5.3.10` 重命名为 `spring-5.3.10-analyse`。

**注意下载后，不能忽略 `spring-aop` 目录下的 `target` 目录。**

## 编译

跳转到源代码目录，执行 `gradle`命令：

```shell
cd spring-5.3.10-analyse/
./gradlew :spring-oxm:compileTestJava
```

出现如下的 `BUILD SUCCESSFUL` 后，用 `Intellij-Idea` 打开项目即可：

```shell
> Task :spring-oxm:xjcGenerate
Errors occurred while build effective model from /Users/vitah/.gradle/caches/modules-2/files-2.1/com.sun.xml.bind/jaxb-xjc/2.2.11/4da3d39ae8ccc39549e3f8971f56035f61d7bf79/jaxb-xjc-2.2.11.pom:
    'dependencyManagement.dependencies.dependency.systemPath' for com.sun:tools:jar must specify an absolute path but is ${tools.jar} in com.sun.xml.bind:jaxb-xjc:2.2.11
Errors occurred while build effective model from /Users/vitah/.gradle/caches/modules-2/files-2.1/com.sun.xml.bind/jaxb-core/2.2.11/db0f76866c6b1e50084e03ee8cf9ce6b19becdb3/jaxb-core-2.2.11.pom:
    'dependencyManagement.dependencies.dependency.systemPath' for com.sun:tools:jar must specify an absolute path but is ${tools.jar} in com.sun.xml.bind:jaxb-core:2.2.11
Errors occurred while build effective model from /Users/vitah/.gradle/caches/modules-2/files-2.1/com.sun.xml.bind/jaxb-impl/2.2.11/2d4b554997fd01d1a2233b1529b22fc9ecc0cf5c/jaxb-impl-2.2.11.pom:
    'dependencyManagement.dependencies.dependency.systemPath' for com.sun:tools:jar must specify an absolute path but is ${tools.jar} in com.sun.xml.bind:jaxb-impl:2.2.11

BUILD SUCCESSFUL in 1m 37s
47 actionable tasks: 47 executed

A build scan was not published as you have not authenticated with server 'ge.spring.io'.
```

## 新建测试模块

`Intellij-Idea` -> `New` -> `Module` ，新建测试模块 `spring-vitahlin`，然后添加所需依赖：

```java
dependencies {
    api(project(":spring-beans"))
    api(project(":spring-core"))
    api(project(":spring-aop"))
    api(project(":spring-context"))
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'
}
```

模块创建完成后，我们还需要创建测试类来验证导入的 spring 源代码能否实现依赖注入

目录结构如下：

```shell
❯ tree -L 2
.
├── BeanScanConfig.java
├── Main.java
└── bean
    └── HelloService.java
```

对应的代码：

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

public class Main {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
		ctx.register(BeanScanConfig.class);
		ctx.refresh();
		HelloService service = ctx.getBean(HelloService.class);
		System.out.println("HelloService" + service);
		service.helloSpring();
	}
}
```

执行 main() 函数，结果如下：

```shell
> Task :spring-vitahlin:Main.main()
HelloServiceorg.springframework.vitahlin.bean.HelloService@6279cee3
Hello Spring!
```
