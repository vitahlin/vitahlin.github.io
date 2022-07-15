---
title: spring源码分析3-核心概念介绍
<!-- description: 这是一个副标题 -->
date: 2022-07-12
slug: introduction-to-spring-framework-core-class
categories:
    - spring

tags:
    - spring
    - java
---

## bean

什么是 `bean`？我们可以来看下 spring 的官方文档：
> In Spring, the objects that form the backbone of your application and that are managed by the Spring IoC container are called beans. A bean is an object that is instantiated, assembled, and otherwise managed by a Spring IoC container. Otherwise, a bean is simply one of many objects in your application. Beans, and the dependencies among them, are reflected in the configuration metadata used by a container.

简而言之，**`bean` 是由 Spring IoC 容器实例化、组装和管理的对象。**

## BeanDefinition

用来表示 `bean` 定义，`BeanDefinition` 中存在很多属性来描述一个 `bean` 的特点。比如：

- class，表示 bean 类型
- scope，表示 bean 作用域，单例或原型等
- lazyInit，表示 bean 是否懒加载
- initMethodName，表示 bean 初始化时要执行的方法
- destoryMethodName，表示 bean 销毁时要执行的方法
- ...

我们可以简单的认为，spring 在扫描class的时候，会根据class文件中的注解等信息，给 `beanDefinition` 对象赋值，所以，`beanDefinition`是对一个类的描述。那spring在真正去初始化、实例化一个bean的时候，直接根据 `beanDefinition` 去初始化就可以，比如，在判断这个bean是根据类型注入，还是根据名字注入的时候，就直接判断 `autowireMode` 的属性值即可。

在 spring 中，我们通常会通过以下几种方式来定义 `bean`：

1. `<bean/>`
2. `@Bean`
3. `@Component`(`@Service`, `@Controller`)

这些，我们可以称之为**声明式定义 `bean`**。

我们还可以编程式定义 `bean`，那就是直接通过 `BeanDefinition`，例如：

```java
public class UserService {
    public void hello(){
        System.out.println("Hello user!");
    }
}


public class BeanDefinitionTest1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
        beanDefinition.setBeanClass(UserService.class);
    
        ctx.registerBeanDefinition("userService", beanDefinition);
        ctx.refresh();
        
        System.out.println(ctx.getBean("userService"));
    }
}
```

运行结果：

```shell
> Task :spring-vitahlin:BeanDefinitionTest1.main()
org.springframework.vitahlin.bean.UserService@67784306
```

我们还可以通过 `BeanDefinition` 设置一个 `bean` 的其他属性：

```java
beanDefinition.setScope("prototype");
beanDefinition.setInitMethodName("init");
beanDefinition.setLazyInit(true);
```

和声明式事务、编程式事务类似，通过 `<bean/>`、`@Bean`、`@Component` 等声明式方式所定义的 bean，最终都会被 spring 解析为对应的 `BeanDefinition` 对象，并放入 spring 容器中。

## BeanDefinitionReader

接下来我们来介绍几种在 Spring 源码中所提供的 `BeanDefinition` 读取器(BeanDefinitionReader)，这些 `BeanDefinitionReader` 在我们使用 Spring 时用的少，但在 spring 源码中用的多，相当于 spring 源码的基础设施。

### AnnotatedBeanDefinitionReader

可以直接把某个类转换为 `BeanDefinition`，例如：

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

import org.springframework.context.annotation.AnnotatedBeanDefinitionReader;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.vitahlin.bean.HelloService;

public class AnnotatedBeanDefinitionReaderTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);
        AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(context);
        
        annotatedBeanDefinitionReader.register(HelloService.class);
        System.out.println(context.getBean("helloService"));
    }
}
```

执行结果：

```shell

> Task :spring-vitahlin:AnnotatedBeanDefinitionReaderTest.main()
org.springframework.vitahlin.bean.HelloService@18a70f16
```

注意：它能解析的注解是：

- @Conditional
- @Scope
- @Lazy
- @Primary
- @DependsOn
- @Role
- @Description

### XmlBeanDefinitionReader

可以解析 `<bean/>` 标签。

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(context);
int i = xmlBeanDefinitionReader.loadBeanDefinitions("spring.xml");
System.out.println(context.getBean("user"));
```

### ClassPathBeanDefinitionScanner

`ClassPathBeanDefinitionScanner` 是扫描器，但是它的作用和 `BeanDefinitionReader` 类似，它可以进行扫描，扫描某个包路径，对扫描到的类进行解析。比如，扫描到的类上如果存在 `@Component` 注解，那么就会把这个类解析为一个 `BeanDefinition`，比如：

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
import org.springframework.context.annotation.ClassPathBeanDefinitionScanner;

public class ClassPathBeanDefinitionScannerTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);
        
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(context);
        scanner.scan("org.springframework.vitahlin.bean");
        
        System.out.println(context.getBean("helloService"));
    }
}
```

运行结果：

```java
> Task :spring-vitahlin:ClassPathBeanDefinitionScannerTest.main()
org.springframework.vitahlin.bean.HelloService@18a70f16
```

## BeanFactory

`BeanFactory` 表示 `Bean` 工厂，`BeanFactory` 会负责创建 `Bean`，并且提供获取 `Bean` 的 API。

`ApplicationContext` 是 `BeanFactory` 的一种，在 spirng 源码中是这么定义的：

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
        ....
        }
```

`ApplicationContext` 继承了 `ListableBeanFactory` 和 `HierarchicalBeanFactory`，而 `ListableBeanFactory` 和 `HierarchicalBeanFactory` 都继承至 `BeanFactory`，所以我们可以认为 ApplicationContext 继承了 BeanFactory。
`ApplicationContext` 拥有了 `BeanFactory` 支持的所有功能，而且比 BeanFactory 更加强大，ApplicationContext 还继承了其他接口，比如 MessageSource国际化，ApplicationEventPublisher 事件发布，EnvironmentCapable 获取环境变量等等。

在 spring 源码实现中，当我们 `new` 一个 `ApplicationContext` 时，其底层会 `new` 一个 `BeanFactory` 出来，当使用 `ApplicationContext` 某些方法时，比如 getBean()，底层调用的是 `BeanFactory` 的 getBean() 方法。

在 spring 源码中，BeanFactory 接口存在一个非常重要的实现类 `DefaultListableBeanFactory`。所以我们也可以直接使用 DefaultListableBeanFactory，而不用使用 ApplicationContext 的某个实现类。例如：

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

import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.vitahlin.bean.HelloService;

public class DefaultListableBeanFactoryTest {
    public static void main(String[] args) {
        
        DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory();
        
        AbstractBeanDefinition abstractBeanDefinition = BeanDefinitionBuilder.genericBeanDefinition()
            .getBeanDefinition();
        abstractBeanDefinition.setBeanClass(HelloService.class);
        
        defaultListableBeanFactory.registerBeanDefinition("helloService", abstractBeanDefinition);
        System.out.println(defaultListableBeanFactory.getBean("helloService"));
    }
}
```

DefaultListableBeanFactory 是非常强大的，支持很多功能，我们可以查看下 DefaultListableBeanFactory 的类继承实现结构：

![DefaultListableBeanFactory类继承实现结构](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207121126755.png)

它实现了很多接口，即拥有很多功能：

1. AliasRegistry 支持别名功能，一个名字可以对应多个别名
2. BeanDefinitionRegistry 可以注册、保存、移除、获取某个BeanDefinition
3. Beanfactory Bean工厂，可以根据某个 Bean 的名字、类型、别名获取某个 Bean 对象
4. SingletonBeanRegistry 可以直接注册、获取某个单例 Bean
5. SimpleAliasRegistry 它是一个类，实现了 AliasRegistry 接口中所定义的功能，支持别名功能
6. ListableBeanFactory 在 BeanFactory 的基础上，增加了其他功能，可以获取所有 BeanDefinition 的 BeanNames，可以根据某个类型获取对应的 BeanNames，可以根据某个类型获取 类型-对应的Bean 的映射关系
7. HierarchicalBeanFactory 在 BeanFactory 的基础上，添加了获取父 BeanFactory 的功能
8. DefaultListableBeanFactory 它是一个类，实现了 SingletonBeanRegistry 接口，拥有了直接注册、获取某个单例 Bean 的功能
9. ConfigurableBeanFactory 在 HierarchicalBeanFactory 和 SingletonBeanRegistry 的基础上，添加了设置父 BeanFactory、类加载器（可以指定某个类加载器进行类的加载）、设置 Spring EL表达式解析器（表示该 BeanFactory可以解析EL表达式）、设置类型转化服务(表示该BeanFactory可以进行类型转化)、可以添加 BeanPostProcessor（表示该BeanFactory支持 Bean 的后置处理器）、可以合并 BeanDefinition、可以销毁某个 Bean 等等功能
10. FactoryBeanRegistrySupport 支持 FactoryBean 的功能
11. AutowireCapableBeanFactory 直接继承了 BeanFactory，在 BeanFactory 的基础上，支持在创建 Bean 的过程中能对 Bean 进行自动装配
12. AbstractBeanFactory 实现了 ConfigurableBeanFactory 接口，继承了 FactoryBeanRegistrySupport，这个 BeanFactory 功能已经很全面，但是不能自动装配和获取 BeanNames。
13. ConfigurableListableBeanFactory 继承了 ListableBeanFactory，实现了 AutowireCapableBeanFactory、ConfigurableBeanFactory
14. AbstractAutowireCapableBeanFactory 继承了 AbstractBeanFactory，实现了 AutowireCapableBeanFactory，拥有自动装配的功能
15. DefaultListableBeanFactory 继承了 AbstractAutowireCapableBeanFactory，实现了 ConfigurableListableBeanFactory 接口和 BeanDefinitionRegistry 接口，所以功能很强大

## ApplicationContext

ApplicationContext 是个接口，实际上也是一个 BeanFactory，不过比 BeanFactory 更加强大，比如：

1. HierarchicalBeanFactory 拥有获取父 BeanFactory 的功能
2. ListableBeanFactory 拥有获取 BeanNames 的功能
3. ResourcePatternResolver 资源加载器，可以一次性获取多个资源（文件资源等）
4. EnvironmentCapable 可以获取运行时环境
5. ApplicationEventPublisher 拥有广播时间等功能
6. MessageSource 拥有国际化功能

我们可以先来看 ApplicationContext 两个比较重要的实现类：

1. AnnotationConfigApplicationContext
2. ClassPathXmlApplicationContext

### AnnotationConfigApplicationContext

类继承实现结构：

![AnnotationConfigApplicationContext类继承实现结构](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207121643845.png)

1. ConfigurableApplicationContext 继承了 ApplicationContext 接口，增加了事件监听器，添加 BeanFactoryPostProcessor、设置 Environment、获取 ConfigurableListableBeanFactory 等功能
2. AbstractApplicationContext 实现了 ConfigurableApplicationContext 接口
3. GenericApplicationContext 继承了 AbstractApplicationContext，实现了 BeanDefinitionRegistry接口，拥有所有 ApplicationContext 的功能，并且可以注册 BeanDefinition，注意这个类中有一个属性 DefaultListableBeanFactory beanFactory
4. AnnotationConfigRegistry 可以单独注册某个类为 BeanDefinition，可以处理该类上的 @Component 注解，可以处理 @Bean 注解，同时可以扫描
5. AnnotationConfigApplicationContext:继承了GenericApplicationContext，实现了 AnnotationConfigRegistry接口，拥有了以上所有的功能

### ClassPathXmlApplicationContext

![ClassPathXmlApplicationContext类继承结构图](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207121658662.png)

它也是继承了AbstractApplicationContext，但是相对于AnnotationConfigApplicationContext 而言，功能没有AnnotationConfigApplicationContext 强大，比如不能注册 BeanDefinition

### 国际化

先定一个 MessageSource：

```java
@Bean  
public MessageSource messageSource() {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    messageSource.setBasename("messages");
    return messageSource;
}
```

有了这个 Bean，就可以在任意想要国际化的地方使用该 MessageSource。同时因为 ApplicationContext 拥有国家化的功能，所以也可以这么用：

```java
context.getMessage("test", null, new Locale("en_CN"))
```

### 资源加载

ApplicationContext 还拥有资源加载的功能，可以直接利用ApplicationContext 获取某个文件的内容：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
Resource resource = context.getResource("file://D:\IdeaProjects\spring‐framework\luban\src\main\java\com\luban\entity\User.java");
System.out.println(resource.contentLength());
```

## BeanPostProcessor

表示 Bean 的后置处理器，可以定义一个或多个 BeanPostProcessor。
一个 BeanPostProcessor 可以在任意一个 Bean 的初始化之前和初始化之后去额外做一些用户自定义的逻辑。我们可以通过判断 beanName 来进行针对性处理（针对某个Bean，或者某部分Bean）。

## BeanFactoryPostProcessor

表示 Bean 工厂的后置处理器，其实和 BeanPostProcessor 类似，BeanPostProcessor 是干涉 Bean 的创建过程，BeanFactoryPostProcessor 是干涉 BeanFactory 的创建过程。

## FactoryBean

我们可以通过 BeanPostProcessor 来干涉 spring 创建 Bean 的过程，但是如果我们想一个 Bean 完完全全由我们来创造，也是可以的，比如通过 `FactoryBean` 接口。

`FactoryBean` 翻译过来是工厂Bean，`BeanFactory` 翻译过来是Bean工厂，`FactoryBean` 是bean工厂`beanFactory` 中的一个 `bean`，只不过这个 `bean` 和一般的 `bean` 不一样，它有着自己的特殊之处，特殊在什么地方呢？**这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似。**

`FactoryBean` 接口源码：

```java
package org.springframework.beans.factory;

import org.springframework.lang.Nullable;

public interface FactoryBean<T> {
    
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
    
    @Nullable
    T getObject() throws Exception;
    
    @Nullable
    Class<?> getObjectType();
    
    default boolean isSingleton() {
        return true;
    }
}
```

`FactoryBean` 接口很简单，就提供了三个方法 `getObject`、`getObjectType`、`isSingleton`。就是这三个方法却成为了spring中很多功能的基础，搜索整个spring的源码可以找到很多 `FactoryBean`，除了spring自身提供的以外，在和一些框架进行集成的时候，同样有 `FactoryBean` 的影子，比如和mybatis集成的时候的 `SqlSessionFactoryBean`。

示例代码：

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

public class Student {
    private String name;
    private String code;
    
    public Student(String name, String code) {
        this.name = name;
        this.code = code;
    }
    
    public Student() {
    }
    
    @Override
    public String toString() {
        return "Student{}";
    }
}
```

```java
package org.springframework.vitahlin.bean;

import org.springframework.beans.factory.FactoryBean;

public class MyFactoryBean implements FactoryBean<Student> {
    
    @Override
    public Student getObject() throws Exception {
        return new Student();
    }
    
    @Override
    public Class<?> getObjectType() {
        return null;
    }
    
    @Override
    public String toString() {
        return "MyFactoryBean{}";
    }
}
```

```java
package org.springframework.vitahlin;

import org.springframework.context.annotation.AnnotatedBeanDefinitionReader;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ClassPathBeanDefinitionScanner;
import org.springframework.vitahlin.bean.HelloService;
import org.springframework.vitahlin.bean.MyFactoryBean;
import org.springframework.vitahlin.bean.Student;

public class FactoryBeanTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);
        AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(context);
        
        annotatedBeanDefinitionReader.register(MyFactoryBean.class);
        
        Student stu = (Student) context.getBean("myFactoryBean");
        
        MyFactoryBean myFactoryBean = (MyFactoryBean) context.getBean("&myFactoryBean");
        System.out.println("stu:" + stu);
        System.out.println("myFactoryBean:" + myFactoryBean);
    }
}
```

运行结果：

```shell
> Task :spring-vitahlin:FactoryBeanTest.main()
stu:Student{}
myFactoryBean:MyFactoryBean{}
```

打印结果很奇怪，通过 `myFactoryBean` 获得了 `Student` 对象，通过 `&myFactoryBean` 获得了 `MyFactoryBean` 对象。为什么会这样？
> 这就是FactoryBean的神奇，通俗点讲 FactoryBean 是一个工厂bean，它是一种可以生产bean的bean，通过其getObejct 方法生产bean。当然不论是 FactoryBean 还是 FactoryBean 生产的 bean 都是受spring管理的，不然通过getBean方法是拿不到的。

`FactroyBean` 的作用是生产一个 `bean`，这里有一个疑问 spring 就是用来生产 `bean` 和管理 `bean` 的，为什么还要有FactoryBean？
> **一般情况下，Spring通过反射机制利用 `<bean>` 的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在 `<bean>` 中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个 `org.springframework.bean.factory.FactoryBean` 的工厂类接口，用户可以通过实现该接口定制实例化bean的逻辑。`FactoryBean` 接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。**

> **FactoryBean的真正目的是让开发者自己去定义那些复杂的bean并交给spring管理，如果bean中要初始化很多变量，而且要进行许多操作，那么使用spring的自动装配是很难完成的，所以spring的开发者把这些工作交给了我们。**

## ExcludeFilter和IncludeFilter

这两个 Filter 是 spring 扫描中用来过滤的。ExcludeFilter 是排除过滤器，IncludeFilter 是包含过滤器。

## MetadataReader、ClassMetadata、AnnotationMetadata

在Spring中需要去解析类的信息，比如类名、类中的方法、类上的注解，这些都可以称之为类的元数 据，所以Spring中对类的元数据做了抽象，并提供了一些工具类。

MetadataReader表示类的元数据读取器，默认实现类为SimpleMetadataReader。
