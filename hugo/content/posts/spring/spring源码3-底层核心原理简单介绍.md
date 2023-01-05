---
title: Spring源码3-底层核心原理简单介绍
<!-- description: 这是一个副标题 -->
date: 2023-01-05
slug: spring-source-code-3
categories:
    - spring

tags:
    - spring
    - java
---


## Spring中的HelloWorld

任何代码的入门都是 `Hello World`  ，同样我们来看下使用 Spring 的入门代码：
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
  
public class HelloWorldTest {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(HelloWorldScanConfig.class);  
        HelloWorldService service = (HelloWorldService) context.getBean("helloWorldService");  
        service.sayHello();  
    }  
}
```

`HelloWorldTest`  中的 main 方法的三行代码，就是我们使用 Spring 的基本方式，可是这几行代码底层到底做了什么？
比如
1. 第一行会构造一个 `AnnotationConfigApplicationContext` 对象，`AnnotationConfigApplicationContext` 对象该如何理解？调用该构造方法除了实例化一个对象还会做哪些事情？
2. 第二行代码，会调用 `AnnotationConfigApplicationContext`  的 `getBean` 方法，会得到一个 `HelloWorldService`  对象，`getBean()` 是如何实现的？返回的 `HelloWorldService`  对象和我们自己直接 `new` 的 `HelloWorldService`  对象有区别吗？
3. 第三行代码比较简单，就是调用 `HelloWorldService`  的 `sayHello()`  方法

光看这三行代码，并不能体现 Spring 的强大之处，但是我们现在可以认为：如果你要用 Spring，你就得这么写。就像要用 Mybatis，就得写各种 Mappe r接口。

接下来，我们就简单介绍一下 Spring 的底层核心原理。

## Spring中是如何创建一个对象？

在Java语言中，肯定是根据某个类来创建一个对象的。

我们再看一下实例代码：
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(HelloWorldScanConfig.class);  
HelloWorldService userService = (HelloWorldService) context.getBean("helloWorldService");  
userService.sayHello();
```

当我们调用 `context.getBean("helloWorldService")`  时，就会去创建一个对象，但是 `getBean`  方法内部怎么知道 "helloWorldService" 对应的是 `HelloWorldService`  类呢？

所以，我们就可以分析出来，在调用 `AnnotationConfigApplicationContext`  的构造方法时，也就是第一行代码，会做这些事情：
1. 解析类 `HelloWorldScanConfig` ，得到扫描路径
2. 遍历扫描路径下的所有类，如果发现某个类上存在 `@Component`、`@Service` 等注解，那么 Spring 就把这个类记录下来，存在一个 `Map` 中，比如 `Map<String, Class>`。（**实际上，Spring源码中确实存在类似的这么一个Map，叫做BeanDefinitionMap**）
3. 3.  Spring 会根据某个规则生成当前类对应的 `beanName` ，作为 key 存入 Map，当前类作为 value

这样，当调用 `context.getBean("helloWorldService")` 时，就可以根据 `helloWorldService` 找到类 `HelloWorldService` ，从而可以创建对象了。

## Bean的创建过程

那么 Spring 中到底是如何创建一个 Bean 的呢，这就是 Bean 的生命周期。大致过程如下：
1. 利用该类的构造方法来创建一个对象（如果该类中有多个构造方法，Spring 则会进行选择，这个叫**推断构造方法**）
2. 得到一个对象后，Spring 会判断该对象是否存在被 `@Autowired` 注解了的属性，把这些属性找出来并由 Spring 进行赋值（**依赖注入**）
3. 依赖注入后，Spring 会判断对象是否实现了 `BeanNameAware` 接口、`BeanClassLoaderAware` 接口、`BeanFactoryAware` 接口，如果实现了，就表示当前对象必须实现该接口中所定义的 `setBeanName()` 、`setBeanClassLoader()`、`setBeanFactory()` 方法，那Spring 就会调用这些方法并传入相应的参数（**Aware回调**）
4. 4.  Aware回调后，Spring 会判断该对象中是否存在某个方法被 `@PostConstruct` 注解了，如果存在，Spring 会调用当前对象的此方法（**初始化前**）
5. 紧接着，Spring 会判断该对象是否实现了 `InitializingBean`  接口，如果实现了，就表示当前对象必须实现该接口中的`afterPropertiesSet()` 方法，那 Spring 就会调用当前对象中的 `afterPropertiesSet()` 方法（**初始化**）
6. 最后，Spring 会判断当前对象是否需要 AOP，如果不需要那么 Bean 就创建完了；如果需要 AOP，则会进行动态代理并生成一个代理对象作为 Bean（**初始化后**）

通过最后一步，我们可以发现，当 Spring 根据 `HelloWorldService` 来创建一个 Bean 时：
1. 如果不用进行 AOP，那么 Bean 就是 `HelloWorldService`  类的构造方法所得到的对象；
2. 如果需要进行 AOP，那么 Bean 就是 `HelloWorldService`  的代理类所实例化得到的对象，而不是 `HelloWorldService`  本身得到的对象。

Bean 对象创建出来后：
1. 如果 Bean 是单例的，那么会把该 Bean 对象存入一个 `Map<String, Object>`，Map 的 key 为 `beanName`，value 为 Bean 对象。这样下次 `getBean` 时就可以直接从 Map 中拿到对应的 Bean 对象了。（实际上，在Spring源码中，这个Map就是**单例池**）
2. 如果当前 Bean 是原型 Bean，那么后续没有其他动作，不回存入 Map，而是在下次 getBean 方法执行时再次执行创建过程，得到一个新的 Bean 对象。

## 推断构造方法

Spring 在基于某个类生成 Bean 的过程中，需要用该类的构造方法来实例化得到一个对象，但是**如果一个类存在多个构造方法，Spring 会使用哪个呢？**

Spring 的判断逻辑如下：
1. 如果一个类只存在一个构造方法，那么不管该构造方法是无参构造方法还是有参构造方法，Spring 都会用这个方法；
2. 如果一个类存在多个构造方法
    - 这些构造方法中，存在一个无参的构造方法，那么 Spring 就会用这个无参的构造方法
    - 这些构造方法中，不存在一个无惨的构造方法，那么 Spring 就会报错

Spring 的设计思想是这样的：
1. 如果一个类只有一个构造方法，那么没得选择，只能用这个构造方法；
2. 如果一个类存在多个构造方法，Spring 不知道如何选择，就会看是否存在无参的构造方法，因为**无参构造方法本身表示一种默认的意义**；
3. 不过如果某个构造方法加了 `@Autowired` 注解，那么就表示程序员告诉 Spring 就用这个加了注解的方法，Spring 就会用这个加了 `@Autowired` 注解的方法

**那么如果 Spring 选择了一个有参的构造方法，在调用构造方法时，需要传入参数，那这个参数怎么来的？**

Spring 会根据入参的类型和入参的名字寻找 Bean 对象（以单例 Bean 为例，Spring 会从单例池那个 Map 中去找）：
1. 先根据入参类型找，如果只找到一个，那就直接用来作为入参；
2. 如果根据入参类型找到多个，则再根据入参名字来确定唯一一个
3. 最终如果没有找到会报错无法创建当前 Bean 对象

***确定用哪个构造方法，确定入参的 Bean 对象，这个过程就叫做推断构造方法。***

## AOP大致流程

AOP 就是进行动态代理。在创建一个 Bean 的过程中，Spring 在最后一步会去判断当前正在创建的这个 Bean 是不是需要进行 AOP，如果需要则会进行动态代理。

如何判断当前 Bean 对象是否需要进行 AOP：
1. 找出所有的切面 Bean；
2. 遍历切面中的每个方法，看是否写了 `@Before`、`@After` 等注解；
3. 如果写了，则判断对应的 `Pointcut` 是否和当前 Bean 对象的类是否匹配
4. 如果匹配则表示当前 Bean 对象有匹配的 `Pointcut` ，需要进行 AOP

利用 cglib 进行 AOP 的大致流程：
1. 生成代理类 `HelloWorldServiceProxy`，代理类继承 `HelloWorldService`；
2. 代理类中重写了父类的方法，比如 `HelloWorldService` 中的 test 方法；
3. 代理类中还会有一个 target 属性，该属性的值为被代理对象（就是通过 `HelloWorldService`  类推断构造方法实例化出来的对象，进行了依赖注入、初始化等步骤的对象）
4. 代理类中的 test 方法被执行时的逻辑如下：
    - 执行切面逻辑（`@Before`） 
    - 调用 `target.test()`

当我们从 Spring 容器得到 `HelloWorldService` 的 Bean 对象时，拿到的就是 `HelloWorldServiceProxy` 所生成的对象，也就是代理对象。

HelloWorldService 代理对象.test()--->执行切面逻辑--->target.test()，注意target对象不是代理对象，而是被代理对象。


## Spring事务

在我们在某个方法上加上了 `@Transactional` 注解后，就表示该方法在调用时会开启 Spring 事务，而这个方法所在的类对应的 Bean 对象会是该类的代理对象。

Spring 事务的代理对象执行某个方法时的步骤：
1. 判断当前执行的方法是否存在 `@Transactional` 注解；
2. 如果存在，则利用事务管理器 `TransactionMananger`  新建一个数据库连接；
3. 修改数据库连接的 `autocommit`  为 false；
4. 执行 `target.test()` ，执行程序所写的业务逻辑代码，也就是执行SQL；
5. 执行完了之后如果没有异常，则提交，否则进行回滚。

Spring 事务是否会失效的判断标准：**某个加了 `@Transactional` 注解的方法被调用时，要判断到底是不是直接被代理对象调用的，如果是则事务会生效，如果不是则失效。**