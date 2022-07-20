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

## 测试代码

新建一个 `AnnotationConfigApplicationContext` 类的使用示例：

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
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
@ComponentScan(basePackages = "org.springframework.vitahlin.bean")  
public class BeanScanConfig {  
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

接下来，我们就来分析 bean `HelloService` 是怎么生成的。

## AnnotationConfigApplicationContext源码

AnnotationConfigApplicationContext源码如下：
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

`register(componentClasses);` 这里则是处理对配置类的注册过程，实际上 bean 的扫描和生成还都是在 `refresh()`  方法中执行，因为这里，我们先了解 bean 的生命周期，所以先着重分析 `refresh()` 流程。

`refresh()` 方法源代码：
```java
@Override  
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        // Prepare this context for refreshing.  
        // 1. 刷新前的准备工作  
        prepareRefresh();  
        // Tell the subclass to refresh the internal bean factory.  
        /**  
         * 告诉子类刷新内部bean工厂  
         * 2. 创建IOC容器DefaultListableBeanFactory，加载解析XML文件（最终存储到Document对象中，读取Document对象，并完成BeanDefinition的加载和注册工作  
         */  
  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
        // Prepare the bean factory for use in this context.  
        // 3. 对IOC容器进行一些预制处理，设置一些公共属性  
        prepareBeanFactory(beanFactory);  
        try {  
            // Allows post-processing of the bean factory in context subclasses.  
            postProcessBeanFactory(beanFactory);  
  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            // Invoke factory processors registered as beans in the context.  
  
            // 这里最终会调用scanner.scan()方法来进行扫描  
            invokeBeanFactoryPostProcessors(beanFactory);  
  
            // Register bean processors that intercept bean creation.  
            registerBeanPostProcessors(beanFactory);  
            beanPostProcess.end();  
  
            // Initialize message source for this context.  
            initMessageSource();  
  
            // Initialize event multicaster for this context.  
            initApplicationEventMulticaster();  
            // Initialize other special beans in specific context subclasses.  
            onRefresh();  
            // Check for listener beans and register them.  
            registerListeners();  
  
            // Instantiate all remaining (non-lazy-init) singletons.  
            // 实例化非懒加载单例bean  
            finishBeanFactoryInitialization(beanFactory);  
            // Last step: publish corresponding event.  
            finishRefresh();  
        } catch (BeansException ex) {  
            if (logger.isWarnEnabled()) {  
                logger.warn("Exception encountered during context initialization - " +  
                    "cancelling refresh attempt: " + ex);  
            }  
            // Destroy already created singletons to avoid dangling resources.  
            destroyBeans();  
            // Reset 'active' flag.  
            cancelRefresh(ex);  
            // Propagate exception to caller.  
            throw ex;  
        } finally {  
            // Reset common introspection caches in Spring's core, since we  
            // might not ever need metadata for singleton beans anymore...            resetCommonCaches();  
            contextRefresh.end();  
        }  
    }  
}
```