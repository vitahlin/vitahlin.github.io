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
> Task :spring-vitahlin:AnnotationConfigApplicationContextTest.main()
org.springframework.vitahlin.bean.HelloService@18a70f16
Hello Spring!
```

接下来，我们就沿着 `AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);` 去看容器上下文的初始化过程。

# AnnotationConfigApplicationContext类初始化

源码如下：

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
    
    private final AnnotatedBeanDefinitionReader reader;
    private final ClassPathBeanDefinitionScanner scanner;

    public AnnotationConfigApplicationContext() {
        StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
        this.reader = new AnnotatedBeanDefinitionReader(this);
        createAnnotatedBeanDefReader.end();
        
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }

    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        register(componentClasses);
        refresh();
    }
}
```

一开始，会执行带参数的构造函数，然后调用无参构造函数。`StartupStep` 是用来统计耗时的，这里不做具体分析。

`this.reader = new AnnotatedBeanDefinitionReader(this);` 这里用于传入指定bean的类的时候使用，会直接进行beanDefinition注册。注意这里传入的是 `this` 参数。实际上，不管是直接指定 `bean` 的类进行注册还是扫描注册，实际上是注册到类 `DefaultListableBeanFactory` 的 `beanDefinitionMap` 参数上，参数类型为 `private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);`，而 `DefaultListableBeanFactory` 是 this 的父类 `GenericApplicationContext` 进行初始化的，因为在Java中子类初始化时会先初始化父类。

```java
public GenericApplicationContext() {
    this.beanFactory = new DefaultListableBeanFactory();
}
```

这里就完成了类的初始化，接下来就开始执行 `register(componentClasses);` 流程。

# componentClasses注册过程

这里的 `componentClasses` 指的是配置类，即 `AnnotationConfigApplicationContext` 构造函数中传入的配置类。

源码如下：

```java
@Override
public void register(Class<?>... componentClasses) {
    Assert.notEmpty(componentClasses, "At least one component class must be specified");
    StartupStep registerComponentClass = this.getApplicationStartup().start("spring.context.component-classes.register")
                .tag("classes", () -> Arrays.toString(componentClasses));
    this.reader.register(componentClasses);
    registerComponentClass.end();
}
```

这里，委派 `AnnotatedBeanDefinitionReader` 进行注册。一层一层调用，最终执行的方法是`AnnotatedBeanDefinitionReader#doRegisterBean`，源码如下：

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
            @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
            @Nullable BeanDefinitionCustomizer[] customizers) {
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }
    // 设置初始化的回调函数，传入一个函数式接口->supplier
    abd.setInstanceSupplier(supplier);
    // 解析scope的值,默认为singleton
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // beanName处理
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    // 解析部分注解: @DependsOn、@Lazy、@Primary、@Role
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // qualifier,多个实现类时，通过这个注解来区分加载的bean
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            } else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            } else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        // 用户自定义BeanDefinition,用于定制化场景
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }
    // 包装BeanDefinition到BeanDefinitionHolder
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 是否需要根据scope生成动态代理对象
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册BeanDefinition
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

到这里，就完成了配置 bean 的注册。

# 组件扫描过程

上文中可以看到 `register` 只是注册了 componentClasses，而基于这些配置类的组件扫描，则在容器的 refresh() 方法中进行处理。
