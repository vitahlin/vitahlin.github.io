---
title: Spring源码1-编译5.3.10源码
<!-- description: 这是一个副标题 -->
date: 2022-07-06
slug: build-spring-source-code
categories:
    - spring

tags:
    - spring
    - java
---


## 源码下载

Github源码地址：[https://github.com/spring-projects/spring-framework/tree/v5.3.10](https://github.com/spring-projects/spring-framework/tree/v5.3.10)下载后将目录 `spring-framework-5.3.10` 重命名为 `spring-5.3.10-analyse`。

**注意gitignore中不能忽略 `spring-aop` 目录下的 `target` 目录。**

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
    api(project(":spring-instrument"))
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'
}
```

模块创建完成后，我们还需要创建测试类来验证导入的 `spring` 源代码能否实现依赖注入。

创建测试类，对应的代码：

```java
package org.springframework.vitahlin.hello;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
@ComponentScan(basePackages = "org.springframework.vitahlin.hello")  
public class HelloWorldScanConfig {  
}
```

```java
package org.springframework.vitahlin.hello;  
  
import org.springframework.stereotype.Service;  
  
@Service  
public class HelloWorldService {  
    public void sayHello() {  
        System.out.println("Hello world");  
    }  
}
```

```java
package org.springframework.vitahlin.hello;  
  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
  
/**  
 * Bean扫描示例  
 */  
public class HelloWorldTest {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(HelloWorldScanConfig.class);  
        HelloWorldService userService = (HelloWorldService) context.getBean("helloWorldService");  
        System.out.println(userService);  
        userService.sayHello();  
    }  
}
```

执行 `main` 函数，结果如下：

```shell
15:44:23: Executing ':spring-vitahlin:HelloWorldTest.main()'...

Starting Gradle Daemon...
Gradle Daemon started in 495 ms
> Task :buildSrc:compileJava UP-TO-DATE
> Task :buildSrc:compileGroovy NO-SOURCE
> Task :buildSrc:pluginDescriptors UP-TO-DATE
> Task :buildSrc:processResources UP-TO-DATE
> Task :buildSrc:classes UP-TO-DATE
> Task :buildSrc:jar UP-TO-DATE
> Task :buildSrc:assemble UP-TO-DATE
> Task :buildSrc:pluginUnderTestMetadata UP-TO-DATE
> Task :buildSrc:compileTestJava NO-SOURCE
> Task :buildSrc:compileTestGroovy NO-SOURCE
> Task :buildSrc:processTestResources NO-SOURCE
> Task :buildSrc:testClasses UP-TO-DATE
> Task :buildSrc:test NO-SOURCE
> Task :buildSrc:validatePlugins UP-TO-DATE
> Task :buildSrc:check UP-TO-DATE
> Task :buildSrc:build UP-TO-DATE
> Task :spring-vitahlin:processResources NO-SOURCE
> Task :spring-expression:processResources UP-TO-DATE
> Task :spring-aop:processResources UP-TO-DATE
> Task :spring-core:cglibRepackJar UP-TO-DATE
> Task :spring-beans:processResources UP-TO-DATE
> Task :spring-core:objenesisRepackJar UP-TO-DATE
> Task :spring-context:processResources UP-TO-DATE
> Task :spring-core:processResources UP-TO-DATE
> Task :spring-instrument:compileJava UP-TO-DATE
> Task :spring-instrument:processResources NO-SOURCE
> Task :spring-instrument:classes UP-TO-DATE
> Task :spring-instrument:jar UP-TO-DATE
> Task :spring-jcl:compileJava UP-TO-DATE
> Task :spring-jcl:processResources UP-TO-DATE
> Task :spring-jcl:classes UP-TO-DATE
> Task :spring-jcl:jar UP-TO-DATE
> Task :spring-core:compileKotlin UP-TO-DATE
> Task :spring-core:compileJava UP-TO-DATE
> Task :spring-core:classes UP-TO-DATE
> Task :spring-core:inspectClassesForKotlinIC UP-TO-DATE
> Task :spring-core:jar UP-TO-DATE
> Task :spring-expression:compileKotlin UP-TO-DATE
> Task :spring-expression:compileJava UP-TO-DATE
> Task :spring-expression:classes UP-TO-DATE
> Task :spring-expression:inspectClassesForKotlinIC UP-TO-DATE
> Task :spring-expression:jar UP-TO-DATE
> Task :spring-beans:compileGroovy UP-TO-DATE
> Task :spring-beans:compileKotlin UP-TO-DATE
> Task :spring-beans:compileJava NO-SOURCE
> Task :spring-beans:classes UP-TO-DATE
> Task :spring-beans:inspectClassesForKotlinIC UP-TO-DATE
> Task :spring-beans:jar UP-TO-DATE
> Task :spring-aop:compileJava UP-TO-DATE
> Task :spring-aop:classes UP-TO-DATE
> Task :spring-aop:jar UP-TO-DATE
> Task :spring-context:compileKotlin
> Task :spring-context:compileJava
> Task :spring-context:compileGroovy NO-SOURCE
> Task :spring-context:classes
> Task :spring-context:inspectClassesForKotlinIC
> Task :spring-context:jar
> Task :spring-vitahlin:compileJava
> Task :spring-vitahlin:classes

> Task :spring-vitahlin:HelloWorldTest.main()
org.springframework.vitahlin.hello.HelloWorldService@c540f5a
Hello world

Deprecated Gradle features were used in this build, making it incompatible with Gradle 8.0.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

See https://docs.gradle.org/7.2/userguide/command_line_interface.html#sec:command_line_warnings

BUILD SUCCESSFUL in 12s
38 actionable tasks: 6 executed, 32 up-to-date

A build scan was not published as you have not authenticated with server 'ge.spring.io'.
15:44:36: Execution finished ':spring-vitahlin:HelloWorldTest.main()'.
```

成功打印地址和输出内容，依赖注入验证成功，但是 gradle 每次都会执行 Task 内容。需要修改 Gradle 的配置，指定 Gradle 为 `Intellij IDEA`，如图所示：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202301041845748.png)

再次执行结果如下：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202301051546795.png)

执行正常，接下来就可以调试spring源代码了。

## 参考
- https://blog.csdn.net/weixin_43405771/article/details/106313378