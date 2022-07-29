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

在spring应用中，通过 `@ComponentScan` 注解可以声明自动注入要扫描的包，spring会将该包下所有加了`@Component` 注解的bean扫描到spring容器中，这里就分析下spring处理bean扫描的源码。

## 测试代码

新建一个 `AnnotationConfigApplicationContext` 类的使用示例：

```java
package org.springframework.vitahlin.bean1;  
  
import org.springframework.stereotype.Component;  
  
@Component  
public class UserService {  
    public void hello() {  
        System.out.println("This is user");  
    }  
}
```

```java
package org.springframework.vitahlin.bean1;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
@ComponentScan(basePackages = "org.springframework.vitahlin.bean1")  
public class BeanScanConfig {  
}
```

```java
package org.springframework.vitahlin;  
  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
import org.springframework.vitahlin.bean1.BeanScanConfig;  
import org.springframework.vitahlin.bean1.UserService;  
  
/**  
 * 注解扫描bean  
 */public class AnnotationConfigApplicationContextTest1 {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);  
        UserService service = (UserService) context.getBean("userService");  
        System.out.println(service);  
        service.hello();  
    }  
}
```

运行结果：
```shell
org.springframework.vitahlin.bean1.UserService@62e136d3
This is user
```

可以看到，上面的代码正确实现了bean的扫描，以及bean的调用，我们就来分析 bean `UserService` 是怎么生成的。

## AnnotationConfigApplicationContext初始化

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

## AbstractApplicationContext的refresh()方法

`AbstractApplicationContext#refresh()` 方法源代码：
```java
@Override  
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        // Prepare this context for refreshing.  
        prepareRefresh();  
        // Tell the subclass to refresh the internal bean factory.  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
        // Prepare the bean factory for use in this context.  
        prepareBeanFactory(beanFactory);  
        try {  
            // Allows post-processing of the bean factory in context subclasses.  
            postProcessBeanFactory(beanFactory);  
  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            // Invoke factory processors registered as beans in the context.  
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

这里要重点关注2个步骤 
- `invokeBeanFactoryPostProcessors(beanFactory)` 
- `finishBeanFactoryInitialization(beanFactory);` 

其中 `invokeBeanFactoryPostProcessors` 方法处理对bean的扫描，`finishBeanFactoryInitialization` 方法处理对非懒加载单例bean的实例化。

`invokeBeanFactoryPostProcessors` 方法处理了 bean 的扫描过程，实际上它会调用 `scanner.scan()` 方法来完成扫描，我们先来看 `scan` 方法的处理逻辑。

## ClassPathBeanDefinitionScanner#scan方法

`ClassPathBeanDefinitionScanner` 源码如下：
```java
private final BeanDefinitionRegistry registry;

public int scan(String... basePackages) {  
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();  
    doScan(basePackages);  
    // Register annotation config processors, if necessary.  
    if (this.includeAnnotationConfig) {  
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);  
    }  
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);  
}
```

可以看到 `scan()` 方法比较简单，传进来包路径作为参数，调用了 `doScan()` 方法来完成扫描处理，值得说明的是 spring 源码中都是类似的命名风格，doXXX 表示具体处理什么。

`ClassPathBeanDefinitionScanner` 是一个扫描器，这里 `registry` 是一个接口 `BeanDefinitionRegistry`，运行的时候实际的值其实就是 `DefaultListableBeanFactory`。但是这里扫描器只需要完成注册 `BeanDefinition` 就可以了，所以没必要用 `DefaultListableBeanFactory`，更符合单一职责。

接下来来看 `ClassPathBeanDefinitionScanner#doScan`  方法的源码：
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Assert.notEmpty(basePackages, "At least one base package must be specified");  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();  
    for (String basePackage : basePackages) {  
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  
  
        for (BeanDefinition candidate : candidates) {  
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);  
    candidate.setScope(scopeMetadata.getScopeName());  
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);  
            if (candidate instanceof AbstractBeanDefinition) {  
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);  
            }  
            if (candidate instanceof AnnotatedBeanDefinition) {  
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);  
            }  
            if (checkCandidate(beanName, candidate)) {  
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);  
                definitionHolder =  
                    AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
                beanDefinitions.add(definitionHolder);  
                registerBeanDefinition(definitionHolder, this.registry);  
            }  
        }  
    }  
    return beanDefinitions;  
}
```

`basePackages` 即指定的扫描路径。
`findCandidateComponents(basePackage);`  这个方法就是最核心的扫描逻辑。

## 默认扫描流程和有配置bean索引的扫描流程

`ClassPathScanningCandidateComponentProvider#findCandidateComponents` ：
```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {  
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {  
		// 有配置bean的索引时用于优化启动速度时，执行这个
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);  
    } else {  
		// 默认情况下，都是执行的这个方法
        return scanCandidateComponents(basePackage);  
    }  
}
```

这个方法会去判断`this.componentsIndex` 是否有值，这个就是我们配置的bean索引，它加载的是`META-INF/spring.components`中的信息，内容格式如下：
```java
com.vitahlin.bean.xxxBean=org.springframework.stereotype.Component
```

配置之后，就会加载这里的Bean组件，不去做扫描，对于项目过于庞大，扫描耗费时间的情况，可以考虑这种方案。一般情况下，都走默认的扫描流程。

## 默认的扫描流程scanCandidateComponents

我们先来了解默认的扫描流程，继续看核心的扫描逻辑 `ClassPathScanningCandidateComponentProvider#scanCandidateComponents`  方法：
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {  
    Set<BeanDefinition> candidates = new LinkedHashSet<>();  
    try {  
        // 1. 根据传进来的包路径构造资源路径  
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;  
        System.out.println("packageSearchPath:" + packageSearchPath);
  
        // 2. 获取basePackage下面的文件资源，找出来的其实就是一个个class文件的file对象  
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);  
  
        boolean traceEnabled = logger.isTraceEnabled();  
        boolean debugEnabled = logger.isDebugEnabled();  
        for (Resource resource : resources) {  
            if (traceEnabled) {  
                logger.trace("Scanning " + resource);  
            }  
            if (resource.isReadable()) {  
                try {  
                    // 3. 某个类的元数据读取器，底层用的ASM技术  
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);  
			        // 4. 判断类是不是一个bean
                    if (isCandidateComponent(metadataReader)) {  
                        // 5.发现是一个bean，就生成一个BeanDefinition  
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
                        sbd.setSource(resource);  
                        // 6.判断类是不是独立的，是不是抽象类，是否有Lookup注解等  
                        if (isCandidateComponent(sbd)) {  
                            if (debugEnabled) {  
                                logger.debug("Identified candidate component class: " + resource);  
                            }  
                            // 符合条件才真正加入结果  
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

## 构建扫描路径

这里，传入包路径就是字符串，先构造对应的包路径，可以添加 `println`  语句来查看 `packageSearchPath` 的值是什么：
```java
String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
    resolveBasePackage(basePackage) + '/' + this.resourcePattern;  
System.out.println("basePackage:" + basePackage);  
System.out.println("packageSearchPath:" + packageSearchPath);
```

结果如下：
```shell
basePackage:org.springframework.vitahlin.bean1
packageSearchPath:classpath*:org/springframework/vitahlin/bean1/**/*.class
```

这里的处理逻辑是包名前面加上`classpath*:`前缀，后面加上`**/*.class`，同时把包名中的`.` 替换成`/` ，得到搜索路径，**最终要匹配查找的是包路径下的 `class`  文件**，包下面可能还有其他文件，但是这里就不管了。

## 获取包路径下资源

拿到对应的包路径后，spring就用本身的resource工具去获取该路径下的资源文件，找出来的其实就是class文件的file对象。调试信息如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207282107211.png)

可以看到，这里的 `Rescource`  就是一个个class文件的文件对象。接下来就是遍历获取到的 `Resource`  资源，逐个处理了。

## 获取类对应的元数据

`getMetadataReader` 则是根据 Resource 获取类对应的元数据。

什么是元数据？
>Metadata is simply data about data. It means it is a description and context of the data. It helps to organize, find and understand data。

上面介绍的大概意思是：元数据是关于数据的数据，元数据是数据的描述和上下文，它有助于组织，查找，理解和使用数据。

spring 对 `MetadataReader` 的描述为:
>Simple facade for accessing class metadata,as read by an ASM.

大意是通过ASM读取class IO流资源组装访问元数据的门面接口。反正，这里通过`MetadataReader` 我们就能拿到类的信息，比如类的名字，类的注解信息等。这里使用了**ASM技术**。

## bean的判断

拿到类的元信息后，就需要判断这个类是不是一个bean。那么这里是怎么处理的呢？
`ClassPathScanningCandidateComponentProvider#isCandidateComponent`  源码：
```java
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {  
    // 类和某个类排除过滤器匹配，就忽略  
    for (TypeFilter tf : this.excludeFilters) {  
        if (tf.match(metadataReader, getMetadataReaderFactory())) {  
            return false;  
        }  
    }  
    // 符合includeFilter的会进行匹配，通过了才是Bean，这里先判断是否有@Component注解  
    for (TypeFilter tf : this.includeFilters) {  
        if (tf.match(metadataReader, getMetadataReaderFactory())) {  
            // 再判断condition条件注解  
            return isConditionMatch(metadataReader);  
        }  
    }  
    return false;  
}
```

**这里并不是直接判断类上的注解，而是去判断类的排除过滤器和包含过滤器。**

- 首先我这个类和任意一个排除过滤器匹配了，那么我就排除这个类。
- 如果排除过滤器符合，那么继续判断包含过滤器。即如果要成为一个bean，一定要符合某个包含过滤器才行。

那么这里为什么是通过判断包含过滤器而不是判断是否有注解`@Component`？
实际上，在**spring的 scanner 初始化的时候会注册一个默认的包含过滤器**，我们来看 scanner 初始化的代码，`ClassPathBeanDefinitionScanner#ClassPathBeanDefinitionScanner` ：
```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,  
                                      Environment environment, @Nullable ResourceLoader resourceLoader) {  
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");  
    this.registry = registry;  
  
    // 注册默认的包含过滤器  
    if (useDefaultFilters) {  
        registerDefaultFilters();  
    }  
    setEnvironment(environment);  
    setResourceLoader(resourceLoader);  
}
```

`ClassPathScanningCandidateComponentProvider#registerDefaultFilters` ：
```java
protected void registerDefaultFilters() {  
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
**方法的第一行即添加了默认的包含过滤器。**
这里判断包含过滤器即判断类上是否有 `@Component` 注解，如果有那么就是匹配的，即是一个bean，再判断条件匹配。


## 构造BeanDifinition

上一步判断完成后，确认类是一个bean，就会根据元数据开始构造一个 `BeanDefinition` 。
```java
ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
```

`ScannedGenericBeanDefinition#ScannedGenericBeanDefinition` ：
```java
public ScannedGenericBeanDefinition(MetadataReader metadataReader) {  
    Assert.notNull(metadataReader, "MetadataReader must not be null");  
    this.metadata = metadataReader.getAnnotationMetadata();  
    // 1. 一开始只是把class name复制给属性，并没有真正的加载类  
    setBeanClassName(this.metadata.getClassName());  
    setResource(metadataReader.getResource());  
}
```


`AbstractBeanDefinition` 源码：
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
注意这里，**一开始我们只是把这个类的名字赋给了BeanDefinition。** 

这里的 beanClass 属性为什么是 Object类型？
>因为在这里，一开始我们只是把bean的名字赋给了它，当我需要这个类的时候，就需要对类进行实例化，实例化后我就能把实例化后的类型赋给beanClass属性，覆盖掉原先的String类型的beanName。所以这个beanClass分两种情况：1. 一开始的时候是类的名字；2. 后面是一个类的对象。


## 判断扫描到的类是不是独立的

```java
if (isCandidateComponent(sbd)) {
	if (debugEnabled) {
		logger.debug("Identified candidate component class: " + resource);
	}
	candidates.add(sbd);
}
```

构造完 `BeanDifinition` 后，这里还会继续判断bean的类型，是不是抽象类，是不是内部类。这里涉及到一个概念，**类是否是独立的**。

注释中是这么说的：
> Determine whether the underlying class is independent, i.e. whether it is a top-level class or a nested class (static inner class) that can be constructed independently of an enclosing class.

具体来看方法`ClassPathScanningCandidateComponentProvider#isCandidateComponent` ：
```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {  
    AnnotationMetadata metadata = beanDefinition.getMetadata();  
  
    // 1. 判断类是不是独立的，处理内部类  
    return (metadata.isIndependent() && (metadata.isConcrete() ||  
        // 确实是抽象类，但是有Lookup注解  
        (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));  
}
```

这里为什么还要再加一个这样的判断呢？
新建一个bean用来测试，ABTestService包含了一个内部类Hello。
```java
package org.springframework.vitahlin.bean1;  
  
import org.springframework.stereotype.Component;  
  
@Component  
public class ABTestService {  
    class Hello {  
    }    
    
    public void hello() {  
        System.out.println("This is ABTestService");  
    }  
}
```

调试效果如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207291611298.png)

可以看到，内部类它也会生成class文件，这里生成class文件 `ABTestService$Hello.class` ，但是它就不能注册 `BeanDefinition` ，这个方法就是用来过滤诸如内部类、抽象类等。即不依赖其他类也能正常被构建，因为后面扫描完成，有一个很重要的步骤，是实例化，而那些非静态的内部类、接口、抽象类是不能被实例化的，故这里排除在外。

判定类是可以注册 `BeanDefinition` 的，那么这里就完成了类的扫描，并且生成了对应的`BeanDefinition` 列表。
扫描完成一个 BeanDefinition 列表后，就开始逐个遍历。

## bean的scope处理 

```java
ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
candidate.setScope(scopeMetadata.getScopeName());
```
这里是处理`BeanDefinition` 的scope属性，默认是单例属性。


## beanDefinition名字生成

```java
String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
```
这行代码，处理了bean名字的生成，这里会调用`AnnotationBeanNameGenerator#generateBeanName` 。

`AnnotationBeanNameGenerator#generateBeanName` 源码：
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
这里分为2部分，先是判断我们是不是在注解上自定义了bean的名字，如果没有则根据默认规则生成bean名字。


### 判断注解是不是定义了bean名字

来看`AnnotationBeanNameGenerator#determineBeanNameFromAnnotation` 方法源码：
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
                // 如果是@Component注解，查看是否有自定义value注解，有的话返回value的值，没有返回null  
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

### 生成默认beanName

如果没有找到我们自定义的beanName，就需要生成一个默认的beanName。

`AnnotationBeanNameGenerator#buildDefaultBeanName` ：
```java
protected String buildDefaultBeanName(BeanDefinition definition) {  
    String beanClassName = definition.getBeanClassName();  
    Assert.state(beanClassName != null, "No bean class name set");  
    String shortClassName = ClassUtils.getShortName(beanClassName);  
    // 1.在这里，如果Bean名称的头两个字符都是大写，那么Bean名称就直接是当前的类名，不然取第一个字母小写的类名  
    return Introspector.decapitalize(shortClassName);  
}
```
`ClassUtils.getShortName` 就是根据bean的全路径名字获取类名，比如 `com.vitahlin.service.HelloSerivce`得到的 `shortClassName` 就是 `HelloService`。

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
这里会判断类名字的第一个字符和第二个字符，**如果前2个字符全部大写的话，那么直接返回类名；否则将返回首字符小写的类名字。** 比如类，`AbcService` 它的bean名字就是 `abcService` ，类 `ABcService` 它的bean名字则是 `ABcService` 。

我们修改测试，来验证上面添加的bean `ABTestService`  是否符合这个规则。
```java
package org.springframework.vitahlin;  
  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
import org.springframework.vitahlin.bean1.ABTestService;  
import org.springframework.vitahlin.bean1.BeanScanConfig;  
import org.springframework.vitahlin.bean1.UserService;  
  
/**  
 * 注解扫描bean  
 */public class AnnotationConfigApplicationContextTest1 {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanScanConfig.class);  
        UserService service = (UserService) context.getBean("userService");  
        System.out.println(service);  
        service.hello();  
        ABTestService abTestService = (ABTestService) context.getBean("ABTestService");  
        System.out.println(abTestService);  
        abTestService = (ABTestService) context.getBean("aBTestService");  
        System.out.println(abTestService);  
    }  
}
```

执行结果：
```shell
org.springframework.vitahlin.bean1.UserService@29ba4338
This is user
org.springframework.vitahlin.bean1.ABTestService@57175e74
Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'aBTestService' available
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeanDefinition(DefaultListableBeanFactory.java:851)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getMergedLocalBeanDefinition(AbstractBeanFactory.java:1362)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:319)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208)
	at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1197)
	at org.springframework.vitahlin.AnnotationConfigApplicationContextTest1.main(AnnotationConfigApplicationContextTest1.java:21)
```

可以看到，bean `ABTestService` 的名字就是 `ABTestService`，我们如果通过 `aBTestService` 去尝试获取的话会直接报错。

## bean的默认值处理

`postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName)` 主要处理一些bean的默认值，比如我们没有设置懒加载选项，就默认设为非懒加载。

## 判断bean是否重复

最终调用的是ClassPathBeanDefinitionScanner#checkCandidate：
```java
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {  
    // 1.判断容器里面是不是已经存在这个beanName  
    if (!this.registry.containsBeanDefinition(beanName)) {  
        return true;  
    }  
  
    BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);  
  
    // 2.这里先不管，后续分析  
    BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();  
    if (originatingDef != null) {  
        existingDef = originatingDef;  
    }  
  
    // 3.判断已经存在的bean和我现在要生成的是不是兼容，如果兼容不抛异常，但是返回false，也不会重新生成  
    if (isCompatible(beanDefinition, existingDef)) {  
        return false;  
    }  
  
    // 4.如果存在相同beanName的bean，这里就会抛出我们平常比较常见的异常，表示beanName冲突  
    throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +  
        "' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +  
        "non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");  
}
```

这里兼容的概念是有可能一个类会被扫描多次，这里就是检查如果是有两个同名的，允许它们是同一个类被扫描了多次，如果已经扫描过，那么不重复注册直接跳过。如果不是兼容的类，但是名字一样就会抛出我们平常比较常见的异常内容。
不重复就将扫描得到的bean添加到列表中。到这里就完成了整个bean扫描的逻辑，接下来就是根据得到的 BeanDefinition 列表进行初始化了。