---
title: spring源码分析4-bean扫描
<!-- description: 这是一个副标题 -->
date: 2022-07-29
slug: spring-bean-scan
categories:
    - spring

tags:
    - spring
    - java
---

## 前言

其实扒开spring的外衣，其实最核心的东西是BeanFactory，spring自己能找到的所有的java bean 全在里面，这个就是整个spring的数据中心。

Spring容器加载方式主要有以下3种：
1. xml配置文件方式 ClassPathXmlApplicationContext
2. 扫描注解方式 AnnotationConfigApplicationContext
3. 本地配置文件 FileSystemXmlApplicationContext

在项目中使用Spring，通过 `@ComponentScan` 注解可以声明自动注入要扫描的包，spring 会将该包下所有加了 `@Component` 注解的 bean 扫描到 spring 容器中，当然spring也可以通过 `xml` 文件来配置要扫描的Bean，不过现在 SpringBoot 一般都是用 @ComponentScan 注解的方式，所以本文主要介绍自动扫描注解的流程和源码。


## 测试用例

首先，我们新建测试代码来实现基本的 Bean 扫描和调用功能。

扫描配置类 BeanScanConfig：
```java
package org.springframework.vitahlin;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
@ComponentScan(basePackages = "org.springframework.vitahlin.bean")  
public class BeanScanConfig {  
}
```

Service Bean: 
```java
package org.springframework.vitahlin.bean;  
  
import org.springframework.stereotype.Service;  
  
@Service  
public class UserService {  
    public void sayHello() {  
        System.out.println("hello userService");  
    }  
}
```

测试的Main函数：
```java
package org.springframework.vitahlin;  
  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
import org.springframework.vitahlin.bean.UserService;  

public class BeanScanTest {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);  
        UserService userService = (UserService) context.getBean("userService");  
        System.out.println(userService);  
        userService.sayHello();  
    }  
}
```

执行结果：
```java
org.springframework.vitahlin.bean.UserService@4206a205
hello userService
```

可以看到，在测试用例中，我们获取到 `userService`  这个bean，并且成功执行了它的方法。

## Bean扫描流程分析

### 容器初始化

首先来看类 `AnnotationConfigApplicationContext`的初始化，`AnnotationConfigApplicationContext` 源码如下：
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

`this.reader = new AnnotatedBeanDefinitionReader(this);` 这里用于传入指定bean的类的时候使用，会直接进行 `BeanDefinition` 注册。注意这里传入的是 `this` 参数。实际上，不管是直接指定 `bean` 的类进行注册还是扫描注册，实际上是注册到类 `DefaultListableBeanFactory` 的 `beanDefinitionMap` 参数上，参数类型为 `private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);`，而 `DefaultListableBeanFactory` 是 this 的父类 `GenericApplicationContext` 进行初始化的，因为在Java中子类初始化时会先初始化父类。

`register(componentClasses);` 这里则是处理对配置类的注册过程，实际上 `bean`  的扫描和生成都是在 `refresh()`  方法中执行，因为这里我们先了解 bean 的扫描流程，所以先着重分析 `refresh()` 流程。

### AbstractApplicationContext#refresh()

```java
@Override  
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        /*** Prepare this context for refreshing. */  
        prepareRefresh();  
        /**  
         * Tell the subclass to refresh the internal bean factory.         * 初始化BeanFactory，解析xml格式的配置文件  
         * xml格式的配置，是在这个方法中扫描到beanDefinitionMap中的  
         * 会把bean.xml解析成一个InputStream，然后再解析成Document格式  
         * 按照Document格式解析，从root节点进行解析，把解析到到信息包装成BeanDefinitionHolder，然后调用DefaultListableBeanFactory  
         * 的注册方法将bean放到beanDefinitionMap中  
         */  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
        // Prepare the bean factory for use in this context.  
        // 准备工厂，给BeanFactory设置属性，添加后置处理器等  
        prepareBeanFactory(beanFactory);  
        try {  
            /*** Allows post-processing of the bean factory in context subclasses. */  
            postProcessBeanFactory(beanFactory);  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            // Invoke factory processors registered as beans in the context.  
            /*** 完成对bean的扫描，将class变为BeaDefinition，并将BeanDefinition存到map中 */  
            invokeBeanFactoryPostProcessors(beanFactory);  
            // Register bean processors that intercept bean creation.  
            registerBeanPostProcessors(beanFactory);  
            beanPostProcess.end();  
            /*** Initialize message source for this context. */  
            initMessageSource();  
            /**  
             * Initialize event multicaster for this context.             * 注册一个多事件派发器  
             * 先从beanFactory获取，如果没有，就创建一个，并将创建的派发器放到beanFactory中  
             */  
            initApplicationEventMulticaster();  
            /**  
             * Initialize other special beans in specific context subclasses.             * 这是一个空方法，在springboot中，如果集成了Tomcat，会在这里new Tomcat，new DispatcherServlert  
             */            onRefresh();  
            /**  
             * Check for listener beans and register them.             * 注册所有的事件监听器  
             * 将容器中的事件监听器添加到applicationEventMulticaster中  
             */  
            registerListeners();  
            /**  
             * Instantiate all remaining (non-lazy-init) singletons.             * 完成对bean的实例化  
             */  
            finishBeanFactoryInitialization(beanFactory);  
            /**  
             * Last step: publish corresponding event.             * 容器刷新完成后，发送容器刷新完成事件  
             */  
            finishRefresh();  
        } catch (BeansException ex) {  
            if (logger.isWarnEnabled()) {  
                logger.warn("Exception encountered during context initialization - " +  
                    "cancelling refresh attempt: " + ex);  
            }  
            /**  
             * Destroy already created singletons to avoid dangling resources.             * 发生异常时，调用bean的销毁方法  
             */  
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

这里要重点关注2个步骤 
1.  `invokeBeanFactoryPostProcessors(beanFactory)`  
2. `finishBeanFactoryInitialization(beanFactory)` 

其中 `invokeBeanFactoryPostProcessors` 方法处理对bean的扫描，`finishBeanFactoryInitialization` 方法处理对非懒加载单例bean的实例化。

`invokeBeanFactoryPostProcessors` 方法处理了 bean 的扫描过程，实际上它会调用 `scanner.scan()` 方法来完成扫描，我们先来看 `scan` 方法的处理逻辑。


### ClassPathBeanDefinitionScanner#scan方法

`ClassPathBeanDefinitionScanner` 源码如下：
```java
private final BeanDefinitionRegistry registry;

public int scan(String... basePackages) {  
    // 看当前容器里面有多少个bean  
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();  
    // 开始扫描  
    doScan(basePackages);  
    // Register annotation config processors, if necessary.  
    if (this.includeAnnotationConfig) {  
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);  
    }  
    // 返回扫描到的bean数量  
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);  
}
```

可以看到 `scan()` 方法比较简单，传进来包路径作为参数，调用了 `doScan()` 方法来完成扫描处理，值得说明的是 spring 源码中都是类似的命名风格，doXXX 表示具体处理什么。

`ClassPathBeanDefinitionScanner` 是一个扫描器，这里 `registry` 是一个接口 `BeanDefinitionRegistry`，运行的时候实际的值其实就是 `DefaultListableBeanFactory`。但是这里扫描器只需要完成注册 `BeanDefinition` 就可以了，所以没必要用 `DefaultListableBeanFactory`，更符合单一职责。

接下来来看 `ClassPathBeanDefinitionScanner#doScan`  方法的源码：
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Assert.notEmpty(basePackages, "At least one base package must be specified");  
    /*** 创建一个空的集合，存放扫描到Bean定义的封装类 */  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();  
    for (String basePackage : basePackages) {  
        /**  
         * 调用父类ClassPathScanningCandidateComponentProvider的方法，扫描给定路径，获取符合条件的Bean定义  
         * 然后遍历扫描得到的Bean  
         */        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  
        for (BeanDefinition candidate : candidates) {  
            /*** 获取Bean定义类中@Scope注解的值，即获取Bean的作用域，然后为Bean设置配置的作用域 */  
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);  
            candidate.setScope(scopeMetadata.getScopeName());  
            /*** 为Bean生成名称 */  
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);  
            /*** 如果扫描得到的Bean不是spring的注解Bean，则为Bean设置默认值，设置Bean的自动注入依赖装配属性等 */  
            if (candidate instanceof AbstractBeanDefinition) {  
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);  
            }  
            /**  
             * 如果扫描得到的Bean是我们的注解Bean，解析@Lazy，@Primary，@DependsOn，@Role，@Description等注解  
             * 然后给beanDefinition设置对应的属性  
             */  
            if (candidate instanceof AnnotatedBeanDefinition) {  
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);  
            }  
            /*** 根据Bean名称检查指定的Bean在容器中是否需要注册，或者有无冲突 */  
            if (checkCandidate(beanName, candidate)) {  
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);  
                /*** 根据Bean中配置的作用域，为Bean应用相应的代理模式 */  
                definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
                beanDefinitions.add(definitionHolder);  
                /*** 向容器注册扫描得到的Bean */  
                registerBeanDefinition(definitionHolder, this.registry);  
            }  
        }  
    }  
    return beanDefinitions;  
}
```

`basePackages` 即指定的扫描路径。
`findCandidateComponents(basePackage)`  这个方法就是最核心的扫描逻辑。

### 默认扫描流程和有配置bean索引的扫描流程选择

ClassPathScanningCandidateComponentProvider#findCandidateComponents:
```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {  
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {  
        // 当项目中bean特别多的时候，可以定义spring.components文件，文件里指定要扫描的bean，可以加快扫描速度，componentsIndex就是索引  
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);  
    } else {  
        // 默认走的这个流程  
        return scanCandidateComponents(basePackage);  
    }  
}
```


#### 有索引的扫描流程

这个方法会去判断`this.componentsIndex` 是否有值，这个就是我们配置的bean索引，它加载的是`META-INF/spring.components`中的信息，内容格式形如：
```java
com.vitahlin.bean.xxxBean=org.springframework.stereotype.Component
```

配置之后，就会加载这里的Bean组件，不去做扫描，对于项目过于庞大，扫描耗费时间的情况，可以考虑这种方案。一般情况下，都走默认的扫描流程。

#### 默认的扫描流程scanCandidateComponents

`ClassPathScanningCandidateComponentProvider#scanCandidateComponents` ：
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {  
    Set<BeanDefinition> candidates = new LinkedHashSet<>();  
    try {  
        /*** 拼接包路径 */  
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;  
        /*** 获取basePackage下面的文件资源，找出来的其实就是一个个class文件的file对象 */  
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);  
        boolean traceEnabled = logger.isTraceEnabled();  
        boolean debugEnabled = logger.isDebugEnabled();  
        /*** 遍历所有的字节码文件，挑选合适的字节码文件生成对应的BeanDefinition */  
        for (Resource resource : resources) {  
            if (traceEnabled) {  
                logger.trace("Scanning " + resource);  
            }  
            if (resource.isReadable()) {  
                try {  
                    /**  
                     * 获取某个类的元数据读取器，元数据读取器可以获取类的名字、注解等信息。  
                     * 比如是不是抽象类，父类是什么。底层采用的是ASM技术。  
                     */  
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);  
                    /*** 判断类是不是一个bean，这里针对的是includeFilters和excludeFilters的匹配校验 */  
                    if (isCandidateComponent(metadataReader)) {  
                        /** 发现是一个bean，就生成一个BeanDefinition，并且把对应的resource都设置进去 */  
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
                        sbd.setSource(resource);  
                        /** 继续判断类的原信息，比如是不是内部类 */  
                        if (isCandidateComponent(sbd)) {  
                            if (debugEnabled) {  
                                logger.debug("Identified candidate component class: " + resource);  
                            }  
                            /** 符合条件才真正加入结果 */  
                            candidates.add(sbd);  
                        } else {  
                            if (debugEnabled) {  
                                logger.debug("Ignored because not a concrete top-level class: " + resource);  
                            }  
                        }  
                    } else {  
                        if (traceEnabled) {  
                            logger.trace("Ignored because not matching any filter: " + resource);  
                        }  
                    }  
                } catch (Throwable ex) {  
                    throw new BeanDefinitionStoreException(  
                        "Failed to read candidate component class: " + resource, ex);  
                }  
            } else {  
                if (traceEnabled) {  
                    logger.trace("Ignored because not readable: " + resource);  
                }  
            }  
        }  
    } catch (IOException ex) {  
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);  
    }  
    return candidates;  
}
```
`scanCandidateComponents` 方法就是扫描的核心流程，

### 构建扫描路径

这里，传入包路径就是字符串，先构造对应的包路径：
```java
String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
    resolveBasePackage(basePackage) + '/' + this.resourcePattern;
```

这里可以加个print语句来查看最终的路径是什么：
```java
String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
    resolveBasePackage(basePackage) + '/' + this.resourcePattern;  
System.out.println("basePackage:" + basePackage);  
System.out.println("packageSearchPath:" + packageSearchPath);
```

执行结果：
```shell
basePackage:org.springframework.vitahlin.bean
packageSearchPath:classpath*:org/springframework/vitahlin/bean/**/*.class
```

这里的处理逻辑是包名前面加上 `classpath*:` 前缀，后面加上 `**/*.class`，同时把包名中的 `.` 替换成 `/` ，得到搜索路径，**最终要匹配查找的是包路径下的 class 文件**，包下面可能还有其他文件，但是这里就不管了。

### 获取包路径下的资源

拿到对应的包路径后，spring 就用本身的 resource 工具去获取该路径下的资源文件，找出来的其实就是 class 文件的 file 对象。调试信息如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210271555723.png)
可以看到，这里的 **Rescource 就是一个个 class 文件的文件对象**，接下来就是遍历获取到的 `Resource` 资源，逐个处理了。

### 获取类对应的元数据

`getMetadataReader` 则是根据 `Resource` 获取类对应的元数据。

什么是元数据？
> Metadata is simply data about data. It means it is a description and context of the data. It helps to organize, find and understand data。

上面介绍的大概意思是：**元数据是关于数据的数据，元数据是数据的描述和上下文，它有助于组织，查找，理解和使用数据。**

spring 对 MetadataReader 的描述为:
> Simple facade for accessing class metadata,as read by an ASM.

大意是通过 ASM 读取 class IO 流资源组装访问元数据的门面接口。反正，这里通过 MetadataReader 我们就能拿到类的信息，比如类的名字，类的注解信息等。这里使用了 ASM 技术。

### 判断是否是@Component组件

**拿到类的元信息后，就需要判断这个类是不是一个 bean**。我们可能定义了一个类，但是这个类并不是我们需要的 Bean，那么这里是怎么处理的呢？ 

> @Component is a generic stereotype for any Spring-managed component. @Repository, @Service, and @Controller are specializations of @Component

大概意思是 `@Component` 是任何Spring管理的组件的通用原型。`@Repository` 、`@Service` 和`@Controller` 是派生自 `@Component` 。

`ClassPathScanningCandidateComponentProvider#isCandidateComponent` 源码：
```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {  
    /** 类和某个类排除过滤器匹配，就忽略 */  
    for (TypeFilter tf : this.excludeFilters) {  
        if (tf.match(metadataReader, getMetadataReaderFactory())) {  
            return false;  
        }  
    }  
    /**  
     * 符合includeFilter的会进行匹配，通过了才是Bean，这里先判断是否有@Component注解。  
     * spring扫描器初始化的时候会注册一个默认的包含过滤器，即类上是否有Component注解的包含过滤器  
     */  
    for (TypeFilter tf : this.includeFilters) {  
        if (tf.match(metadataReader, getMetadataReaderFactory())) {  
            /** 再判断condition条件注解 */  
            return isConditionMatch(metadataReader);  
        }  
    }  
    return false;  
}
```

这里并不是直接判断类上的注解，而是**去判断类的排除过滤器和包含过滤器**：
1. 首先我这个类和任意一个排除过滤器匹配了，那么我就排除这个类
2. 如果排除过滤器符合，那么继续判断包含过滤器。即如果要成为一个 bean，一定要符合某个包含过滤器才行。

那么这里为什么是通过判断包含过滤器而不是判断是否有注解 `@Component` ？

实际上，在 **spring 的 scanner 初始化的时候会注册一个默认的包含过滤器**，我们来看 `ClassPathBeanDefinitionScanner`  初始化的代码，`ClassPathBeanDefinitionScanner#ClassPathBeanDefinitionScanner` ：
```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,  
                                      Environment environment, @Nullable ResourceLoader resourceLoader) {  
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");  
    this.registry = registry;  
    /*** 注册默认的包含过滤器 */  
    if (useDefaultFilters) {  
        registerDefaultFilters();  
    }  
    setEnvironment(environment);  
    setResourceLoader(resourceLoader);  
}
```

继续跟进方法 `registerDefaultFilters` ，`ClassPathScanningCandidateComponentProvider#registerDefaultFilters`：
```java
protected void registerDefaultFilters() {  
    // 添加默认的Component注解包含过滤器  
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));  
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();  
    try {  
        this.includeFilters.add(new AnnotationTypeFilter(  
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));  
        logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");  
    } catch (ClassNotFoundException ex) {  
        // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.  
    }  
    try {  
        this.includeFilters.add(new AnnotationTypeFilter(  
            ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));  
        logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");  
    } catch (ClassNotFoundException ex) {  
        // JSR-330 API not available - simply skip.  
    }  
}
```
**方法的第一行即添加了默认的包含过滤器**。 

我们可以新建一个非bean的类来测试下。

#### 非Bean类测试用例 

新建一个类Student，但是不加 `@Component`  注解：
```java
package org.springframework.vitahlin.bean;  
  
public class Student {  
    public void hello() {  
        System.out.println("hello Student");  
    }  
}
```

然后开始断点：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210271938365.png)
图中可以看到，当前 `for`  循环处理的是 `Student.class`  这个类，然后右键 `isCandidateComponent`  -> `Evuluate Expression` ，得到的结果为 false，即 Spring 不认为 Student 是一个 bean。
所以这里判断包含过滤器即判断类上是否有 @Component 注解，如果有那么就是匹配的，即是一个 bean，再判断条件匹配。

### 构造 BeanDefinition

上一步判断完成后，确认类是一个bean，就会根据元数据开始构造一个 `BeanDefinition` 。
```java
ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
```

跟进 `ScannedGenericBeanDefinition` 的构造函数`ScannedGenericBeanDefinition#ScannedGenericBeanDefinition` ：
```java
public ScannedGenericBeanDefinition(MetadataReader metadataReader) {  
    Assert.notNull(metadataReader, "MetadataReader must not be null");  
    this.metadata = metadataReader.getAnnotationMetadata();  
    /*** 一开始只是把class name复制给属性，并没有真正的加载类 */  
    setBeanClassName(this.metadata.getClassName());  
    setResource(metadataReader.getResource());  
}
```

`AbstractBeanDefinition#setBeanClassName` 方法处理：
```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {
	@Nullable
	private volatile Object beanClass;
	
	@Override
	public void setBeanClassName(@Nullable String beanClassName) {
		this.beanClass = beanClassName;
	}
}
```

注意这里，**一开始我们只是把这个类的名字赋给了BeanDefinition**。

这里的 `beanClass` 属性为什么是 `Object` 类型？
>因为在这里，一开始我们只是把bean的名字赋给了它，当我需要这个类的时候，就需要对类进行实例化，实例化后我就能把实例化后的类型赋给 beanClass 属性，覆盖掉原先的String类型的 BeanName。所以这个 beanClass 分两种情况：
> 1. 一开始的时候是类的名字
> 2. 后面实例化后一个类的对象。

### 判断扫描到的类是不是独立的

```java
if (isCandidateComponent(sbd)) {
	if (debugEnabled) {
		logger.debug("Identified candidate component class: " + resource);
	}
	candidates.add(sbd);
}
```

构造完 `BeanDefinition` 后，这里还会继续判断bean的类型，比如是不是抽象类。这里涉及到一个概念，**类是否是独立的**。

源码注释中是这么说的：
> Determine whether the underlying class is independent, i.e. whether it is a top-level class or a nested class (static inner class) that can be constructed independently of an enclosing class.

具体来看方法 `ClassPathScanningCandidateComponentProvider#isCandidateComponent`：
```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {  
    AnnotationMetadata metadata = beanDefinition.getMetadata();  
    /**  
     * isIndependent当前类是否顶级类或者嵌套类  
     * isConcrete不是接口，不是抽象类，就返回true  
     * 如果bean是抽象类并且添加类@Lookup注解，那么可以注入  
     */  
    return (metadata.isIndependent() && (metadata.isConcrete() ||  
        // 确实是抽象类，但是有Lookup注解  
        (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));  
}
```

#### 测试用例

这里涉及到了独立类的概念，新建 `UserInterface` 类：
```java
package org.springframework.vitahlin.bean;  
  
import org.springframework.stereotype.Service;  
  
@Service  
public interface UserInterface {  
    void sayHello();  
}
```

然后断点查看isCandidateComponent的执行结果，如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210272012931.png)

虽然 `UserInterface` 加 了 `@Service`  注解，但是它并不是独立的将会被过滤。这个方法就是用来过滤诸如内部类、抽象类等，即不依赖其他类也能正常被构建。因为后续扫描完成，有一个很重要的步骤是实例化，而那些非静态的内部类、接口、抽象类是不能被实例化的，故这里需排除在外。

反之，对 `UserService` 的判定：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210272015004.png)

可以看到，结果为 `true` ，`UserService` 可以作为bean处理。

判定类是可以注册 `BeanDefinition` 的，那么这里就完成了类的扫描，并且生成了对应的 `BeanDefinition` 列表。
接下来就就开始逐个遍历扫描得到的 `BeanDefinition` ，根据 `BeanDefinition`  生成对应的 Bean。

### Bean的scope处理

```java
ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
sbd.setSource(resource);
```
这里是处理 `BeanDefinition`的`scope`属性，默认是单例属性。

### BeanDefinition名字生成

处理完 `BeanDefinition` 属性后，开始处理Bean的名字：
```java
String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
```

这行代码，处理了bean名字的生成，这里会调用 `AnnotationBeanNameGenerator#generateBeanName`，根据`AnnotationBeanNameGenerator#generateBeanName` 源码：
```java
@Override  
public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {  
    if (definition instanceof AnnotatedBeanDefinition) {  
        String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);  
        if (StringUtils.hasText(beanName)) {  
            // Explicit bean name found.  
            return beanName;  
        }  
    }  
    // Fallback: generate a unique default bean name.  
    // 没有找到@Component注解自定义的value值，则生成一个  
    return buildDefaultBeanName(definition, registry);  
}
```

这里分为2部分：
1.  先是判断我们是不是在注解上自定义了bean的名字
2.  如果没有则根据默认规则生成bean名字

#### 判断注解是不是定义了bean名字

`AnnotationBeanNameGenerator#determineBeanNameFromAnnotation` 方法源码：
```java
@Nullable  
protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {  
    AnnotationMetadata amd = annotatedDef.getMetadata();  
    Set<String> types = amd.getAnnotationTypes();  
    String beanName = null;  
    for (String type : types) {  
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);  
        if (attributes != null) {  
            Set<String> metaTypes = this.metaAnnotationTypesCache.computeIfAbsent(type, key -> {  
                Set<String> result = amd.getMetaAnnotationTypes(key);  
                return (result.isEmpty() ? Collections.emptySet() : result);  
            });  
            if (isStereotypeWithNameValue(type, metaTypes, attributes)) {  
                /*** 如果是@Component注解，查看是否有自定义value注解，有的话返回value的值，没有返回null */  
                Object value = attributes.get("value");  
                if (value instanceof String) {  
                    String strVal = (String) value;  
                    if (StringUtils.hasLength(strVal)) {  
                        if (beanName != null && !strVal.equals(beanName)) {  
                            throw new IllegalStateException("Stereotype annotations suggest inconsistent " +  
                                "component names: '" + beanName + "' versus '" + strVal + "'");  
                        }  
                        beanName = strVal;  
                    }  
                }  
            }  
        }  
    }  
    return beanName;  
}
```

获取当前类上的注解，然后 `isStereotypeWithNameValue` 方法判断是否存在注解 `@Component`，如果有的话获取它的 `value` 值，如果没有注解或者没有 `value` 值，则返回 `null`。

#### 生成默认的Bean名称

如果没有找到我们自定义的 beanName，就需要生成一个默认的 beanName，`AnnotationBeanNameGenerator#buildDefaultBeanName` ：
```java
protected String buildDefaultBeanName(BeanDefinition definition) {  
    String beanClassName = definition.getBeanClassName();  
    Assert.state(beanClassName != null, "No bean class name set");  
    String shortClassName = ClassUtils.getShortName(beanClassName);  
    /*** 在这里，如果Bean名称的头两个字符都是大写，那么Bean名称就直接是当前的类名，不然取第一个字母小写的类名 */  
    return Introspector.decapitalize(shortClassName);  
}
```

`ClassUtils.getShortName`  就是根据bean的全路径名字获取类名，比如 `com.vitahlin.service.HelloSerivce` 得到的 `shortClassName` 就是 `HelloService`。

这里，重点来看`Introspector#decapitalize` ：
```java
public static String decapitalize(String name) {  
    if (name == null || name.length() == 0) {  
        return name;  
    }  
    if (name.length() > 1 && Character.isUpperCase(name.charAt(1)) &&  
                    Character.isUpperCase(name.charAt(0))){  
        return name;  
    }  
    char chars[] = name.toCharArray();  
    chars[0] = Character.toLowerCase(chars[0]);  
    return new String(chars);  
}
```
会判断类名字的第一个字符和第二个字符，**如果前2个字符全部大写的话，那么直接返回类名；否则将返回首字符小写的类名字。** 比如类，`AbcService` 它的bean名字就是 `abcService` ，类 `ABcService` 它的bean名字则是 `ABcService` 。

### Bean的默认值处理

```java
/*** 如果扫描得到的Bean不是spring的注解Bean，则为Bean设置默认值，设置Bean的自动注入依赖装配属性等 */  
if (candidate instanceof AbstractBeanDefinition) {  
    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);  
}
```
主要处理一些bean的默认值，比如我们没有设置懒加载选项，就默认设为非懒加载。

### 判断Bean是否重复

接下来，方法 `checkCandidate(beanName, candidate)` 判断类是否重复，最终调用的是`ClassPathBeanDefinitionScanner#checkCandidate`：
```java
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {  
    /*** 判断容器里面是不是已经存在这个beanName */  
    if (!this.registry.containsBeanDefinition(beanName)) {  
        return true;  
    }  
    BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);  
    /*** 这里先不分析，后续分析 */  
    BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();  
    if (originatingDef != null) {  
        existingDef = originatingDef;  
    }  
    /**  
     * 判断已经存在的bean和我现在要生成的是不是兼容，  
     * 如果兼容不抛异常，但是返回false，也不会重新注册bean  
     */    if (isCompatible(beanDefinition, existingDef)) {  
        return false;  
    }  
    /**  
     * 如果存在相同beanName的bean，  
     * 这里就会抛出我们平常比较常见的异常，表示beanName冲突  
     */  
    throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +  
        "' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +  
        "non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");  
}
```

兼容的概念是有可能一个类会被扫描多次，这里就是检查如果是有两个同名的，允许它们是同一个类被扫描了多次，如果已经扫描过，那么不重复注册直接跳过。如果不是兼容的类，但是名字一样就会抛出我们平常比较常见的异常内容。

不重复就将扫描得到的bean添加到列表中。到这里就完成了整个bean扫描的逻辑，接下来就是根据得到的 `BeanDefinition` 列表进行初始化了。

### 向容器注册扫描得到的BeanDefinition

```java
registerBeanDefinition(definitionHolder, this.registry);
```

这个方法调用的是 `BeanDefinitionReaderUtils#registerBeanDefinition` ：
```java
public static void registerBeanDefinition(  
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)  
    throws BeanDefinitionStoreException {  
    // Register bean definition under primary name.  
    String beanName = definitionHolder.getBeanName();  
    // 注册到bean工厂  
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());  
    // Register aliases for bean name, if any.  
    String[] aliases = definitionHolder.getAliases();  
    if (aliases != null) {  
        for (String alias : aliases) {  
            registry.registerAlias(beanName, alias);  
        }  
    }  
}
```

看其中的 `registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())`，它是一个接口 `BeanDefinitionRegistry`，实际上的实现方法是`DefaultListableBeanFactory#registerBeanDefinition`：
```java
@Override  
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)  
    throws BeanDefinitionStoreException {  
    Assert.hasText(beanName, "Bean name must not be empty");  
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");  
    if (beanDefinition instanceof AbstractBeanDefinition) {  
        try {  
            ((AbstractBeanDefinition) beanDefinition).validate();  
        } catch (BeanDefinitionValidationException ex) {  
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,  
                "Validation of bean definition failed", ex);  
        }  
    }  
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);  
    if (existingDefinition != null) {  
        if (!isAllowBeanDefinitionOverriding()) {  
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);  
        } else if (existingDefinition.getRole() < beanDefinition.getRole()) {  
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE  
            if (logger.isInfoEnabled()) {  
                logger.info("Overriding user-defined bean definition for bean '" + beanName +  
                    "' with a framework-generated bean definition: replacing [" +  
                    existingDefinition + "] with [" + beanDefinition + "]");  
            }  
        } else if (!beanDefinition.equals(existingDefinition)) {  
            if (logger.isDebugEnabled()) {  
                logger.debug("Overriding bean definition for bean '" + beanName +  
                    "' with a different definition: replacing [" + existingDefinition +  
                    "] with [" + beanDefinition + "]");  
            }  
        } else {  
            if (logger.isTraceEnabled()) {  
                logger.trace("Overriding bean definition for bean '" + beanName +  
                    "' with an equivalent definition: replacing [" + existingDefinition +  
                    "] with [" + beanDefinition + "]");  
            }  
        }  
        // 这里把beanDefinition放到beanMap里面  
        this.beanDefinitionMap.put(beanName, beanDefinition);  
    } else {  
        if (hasBeanCreationStarted()) {  
            // Cannot modify startup-time collection elements anymore (for stable iteration)  
            synchronized (this.beanDefinitionMap) {  
                this.beanDefinitionMap.put(beanName, beanDefinition);  
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);  
                updatedDefinitions.addAll(this.beanDefinitionNames);  
                updatedDefinitions.add(beanName);  
                this.beanDefinitionNames = updatedDefinitions;  
                removeManualSingletonName(beanName);  
            }  
        } else {  
            // Still in startup registration phase  
            this.beanDefinitionMap.put(beanName, beanDefinition);  
            this.beanDefinitionNames.add(beanName);  
            removeManualSingletonName(beanName);  
        }  
        this.frozenBeanDefinitionNames = null;  
    }  
    if (existingDefinition != null || containsSingleton(beanName)) {  
        resetBeanDefinition(beanName);  
    } else if (isConfigurationFrozen()) {  
        clearByTypeCache();  
    }  
}
```

其中
-   `this.beanDefinitionMap.put(beanName, beanDefinition)`这行就是把扫描得到的 BeanDefinition 注册到集合中
-   `this.beanDefinitionNames.add(beanName);`这里还会保存一份 BeanName的集合。

至此就完成了Bean 的扫描整个流程，容器中已经有了对应的 `BeanDefinition`，后续根据这些 `BeanDefinition` 进行实例化。

## 总结

这里仅分析了通用的Bean的扫描，`FactoryBean`  的处理会在另外的文中分析。