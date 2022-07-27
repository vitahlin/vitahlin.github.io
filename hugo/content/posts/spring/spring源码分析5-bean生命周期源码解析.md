---
title: spring源码分析5-bean生命周期源码分析1
<!-- description: 这是一个副标题 -->
date: 2022-07-14
slug: spring-bean-life-cycle1
categories:
    - spring

tags:
    - spring
    - java
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

`AnnotationConfigApplicationContext` 源码如下：
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

`register(componentClasses);` 这里则是处理对配置类的注册过程，实际上 bean 的扫描和生成还都是在 `refresh()`  方法中执行，因为这里我们先了解 bean 的生命周期，所以先着重分析 `refresh()` 流程。

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


这里重点关注2个步骤 `invokeBeanFactoryPostProcessors(beanFactory)` 和 `finishBeanFactoryInitialization(beanFactory);` 其中 `invokeBeanFactoryPostProcessors` 处理对bean的扫描，`finishBeanFactoryInitialization` 处理对非懒加载单例bean的实例化。


## bean的扫描

`invokeBeanFactoryPostProcessors` 方法处理了 bean 的扫描过程，实际上它会调用 `scanner.scan()` 方法来完成扫描。这里我们先来看 `scan` 方法的处理逻辑，后面再来分析 `invokeBeanFactoryPostProcessors` 的具体处理。


### ClassPathBeanDefinitionScanner#scan方法

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

接下来来看 ClassPathBeanDefinitionScanner#doScan 方法的源码：
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
`Set<BeanDefinition> candidates = findCandidateComponents(basePackage);` 就行就是最核心的扫描逻辑。

ClassPathScanningCandidateComponentProvider#findCandidateComponents：
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

继续看核心的扫描逻辑 ClassPathScanningCandidateComponentProvider#scanCandidateComponents 方法：
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

### 构建扫描路径

这里，我们传入包路径就是字符串，构造对应的包路径，在这里可以添加 `println`  语句来打印 packageSearchPath 的值。结果如下：`packageSearchPath:classpath*:org/springframework/vitahlin/bean/**/*.class` 。可以看到，最终要匹配查找的是包路径下的 `class`  文件，包下面可能还有其他文件，但是这里就不管了。

### 获取包路径下资源

拿到对应的包路径后，spring就用本身的resource工具去获取该路径下的资源文件，找出来的其实就是class文件的file对象。调试信息如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207251156802.png)

可以看到，这里的 rescource 就是一个个class文件的文件对象。

### bean的判定

然后开始遍历 resource，会去得到这个类的对应的**元数据读取器**，我们用这个元数据读取器就可以得到这个类的一些信息和注解信息，比如类的名字、是不是抽象类等。这里**底层采用的是ASM技术**。

拿到类的信息后，就需要判断这个类是不是一个bean。那么这里是怎么处理的呢？
ClassPathScanningCandidateComponentProvider#isCandidateComponent 源码：
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

这里并不是直接判断类上的注解，而是去判断类的排除过滤器和包含过滤器。
- 首先我这个类和任意一个排除过滤器匹配了，那么我就排除这个类。
- 如果排除过滤器符合，那么继续判断包含过滤器。如果要成为一个bean，一定要符合某个包含过滤器才行。实际上，在spring的 scanner 初始化的时候会注册一个默认的包含过滤器，ClassPathBeanDefinitionScanner#ClassPathBeanDefinitionScanner：
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

ClassPathScanningCandidateComponentProvider#registerDefaultFilters：
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


### 构造beanDifinition

上一步判定完成后，确认类是一个bean，就会开始构造一个 beanDefinition。
```java
ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
```

ScannedGenericBeanDefinition#ScannedGenericBeanDefinition：
```java
public ScannedGenericBeanDefinition(MetadataReader metadataReader) {  
    Assert.notNull(metadataReader, "MetadataReader must not be null");  
    this.metadata = metadataReader.getAnnotationMetadata();  
    // 1. 一开始只是把class name复制给属性，并没有真正的加载类  
    setBeanClassName(this.metadata.getClassName());  
    setResource(metadataReader.getResource());  
}
```


AbstractBeanDefinition源码：
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
注意这里，**一开始我们只是把这个类的名字赋给了这个beanDefinition。**

这里的 beanClass 属性为什么是 Object类型？
>因为在这里，一开始我们只是把bean的名字赋给了它，当我需要这个类的时候，就需要对类进行实例化，实例化后我就能把实例化后的类型赋给beanClass属性，覆盖掉原先的String类型的beanName。所以这个beanClass分两种情况：1. 一开始的时候是类的名字；2. 后面是一个类的对象。


### 判断beanDefinition是不是内部类，是不是抽象类

构造完 beanDifinition后，这里还会继续判断bean的类型，判断是不是抽象类，是不是内部类。
```java
if (isCandidateComponent(sbd)) {
	if (debugEnabled) {
		logger.debug("Identified candidate component class: " + resource);
	}
	candidates.add(sbd);
}
```

具体来看方法ClassPathScanningCandidateComponentProvider#isCandidateComponent：
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
我们可以修改HelloService的代码，新建一个内部类用来测试。
```java
package org.springframework.vitahlin.bean;  
  
import org.springframework.stereotype.Component;  
  
@Component  
public class HelloService {  
    class Hello {  
    }    public void helloSpring() {  
        System.out.println("Hello Spring!");  
    }  
}
```

调试效果如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202207251753480.png)
可以看到，内部类它也会生成class文件，但是它就不能注册 `BeanDefinition` ，所有这个方法就是用来过滤诸如内部类、抽象类等。

判定类是可以注册 `BeanDefinition` 的，那么这里就完成了类的扫描，并且生成了对应的`BeanDefinition` 列表。
扫描完成一个 BeanDefinition 列表后，就开始逐个遍历。

### bean的scope处理 

```java
ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
candidate.setScope(scopeMetadata.getScopeName());
```
这里是处理beanDefinition的scope属性。


### beanDefinition名字生成

```java
String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
```
这行代码，处理了bean名字的生成，这里会调用AnnotationBeanNameGenerator#generateBeanName。

AnnotationBeanNameGenerator#generateBeanName
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


#### 判断注解是不是定义了bean名字

来看AnnotationBeanNameGenerator#determineBeanNameFromAnnotation方法源码：
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

AnnotationBeanNameGenerator#buildDefaultBeanName：
```java
protected String buildDefaultBeanName(BeanDefinition definition) {  
    String beanClassName = definition.getBeanClassName();  
    Assert.state(beanClassName != null, "No bean class name set");  
    String shortClassName = ClassUtils.getShortName(beanClassName);  
    // 1.在这里，如果Bean名称的头两个字符都是大写，那么Bean名称就直接是当前的类名，不然取第一个字母小写的类名  
    return Introspector.decapitalize(shortClassName);  
}
```
ClassUtils.getShortName就是根据bean的全路径名字获取类名，比如 `com.vitahlin.service.HelloSerivce`得到的 `shortClassName` 就是 `HelloService`。

这里，重点来看Introspector#decapitalize：
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


### bean的默认值处理

`postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName)` 主要处理一些bean的默认值，比如我们没有设置懒加载选项，就默认设为非懒加载。

### 判断bean是否重复

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
会从当前的bean列表里面判断是否存在相同名字的bean，如果已经存在就会抛出我们平常比较常见的异常内容。不重复就将扫描得到的bean添加到map中。到这里就完成了整个bean扫描的逻辑，我们会有一个map，里面有一系列的 BeanDefinition 对象。接下来就是根据得到的 BeanDefinition 列表进行初始化了。

## bean的生成

在`AbstractApplicationContext#refresh()` 中，处理bean生成的逻辑是 `finishBeanFactoryInitialization(beanFactory);` ，我们一步一步看下去，最终执行的代码是`DefaultListableBeanFactory#preInstantiateSingletons` ，源码如下：
```java
@Override  
public void preInstantiateSingletons() throws BeansException {  
    if (logger.isTraceEnabled()) {  
        logger.trace("Pre-instantiating singletons in " + this);  
    }  
    // Iterate over a copy to allow for init methods which in turn register new bean definitions.  
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.    // 取出所有的bean名字  
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);  
    // Trigger initialization of all non-lazy singleton beans...  
    for (String beanName : beanNames) {  
  
        /**  
         * 1.获取合并后的bean，自己本身是一个beanDefinition，如果有父beanDefinition，那么就是父beanDefinition和当前beanDefinition合并后生成一个  
         * 新的rootBeanDefinition  
         */        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);  
  
        /**  
         * 2.判断当前bean是不是单例、是不是非懒加载  
         * isAbstract判断当前beanDefinition是不是抽象的，跟抽象类没有关系  
         * 抽象的beanDefinition主要是给其他的beanDefinition当父亲的bean  
         */        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {  
            if (isFactoryBean(beanName)) {  
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
                if (bean instanceof FactoryBean) {  
                    FactoryBean<?> factory = (FactoryBean<?>) bean;  
                    boolean isEagerInit;  
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {  
                        isEagerInit = AccessController.doPrivileged(  
                            (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,  
                            getAccessControlContext());  
                    } else {  
                        isEagerInit = (factory instanceof SmartFactoryBean &&  
                            ((SmartFactoryBean<?>) factory).isEagerInit());  
                    }  
                    if (isEagerInit) {  
                        getBean(beanName);  
                    }  
                }  
            } else {  
                // 3.创建bean  
                getBean(beanName);  
            }  
        }  
    }  
  
    // 到这里，单例bean全部创建完成  
  
    // Trigger post-initialization callback for all applicable beans...  
    for (String beanName : beanNames) {  
        Object singletonInstance = getSingleton(beanName);  
    /**  
         * 判断单例bean是不是实现了SmartInitializingSingleton接口  
         * 所有的单例bean都创建后再调用SmartInitializingSingleton  
         */        if (singletonInstance instanceof SmartInitializingSingleton) {  
            StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")  
                .tag("beanName", beanName);  
            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;  
            if (System.getSecurityManager() != null) {  
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {  
                    smartSingleton.afterSingletonsInstantiated();  
                    return null;                }, getAccessControlContext());  
            } else {  
                smartSingleton.afterSingletonsInstantiated();  
            }  
            smartInitialize.end();  
        }  
    }  
}
```
这里，就是遍历我们上一步保存的bean名字列表，初始化每个bean。

### bean的合并

首先就是要处理bean的合并`getMergedLocalBeanDefinition` ，这里涉及到了**bean的继承**，我们先用代码来演示下bean的继承：
```java
package org.springframework.vitahlin.bean;  
  
public class AmericaBean {  
}
```

```java
package org.springframework.vitahlin.bean;  
  
public class ChinaBean {  
}
```

```java
package org.springframework.vitahlin.bean;  
  
public class BeijingBean {  
}
```

```java
package org.springframework.vitahlin.bean;  
  
public class CountryBean {  
}
```



```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">  
  
    <bean id="countryBean" class="org.springframework.vitahlin.bean.CountryBean" scope="prototype" abstract="true">  
    </bean>  
    <bean id="chinaBean" class="org.springframework.vitahlin.bean.ChinaBean" scope="singleton" parent="countryBean">  
    </bean>  
    <bean id="americaBean" class="org.springframework.vitahlin.bean.AmericaBean" parent="countryBean">  
    </bean>  
    <bean id="beijingBean" class="org.springframework.vitahlin.bean.BeijingBean" scope="singleton" parent="chinaBean">  
    </bean>  
</beans>
```

```java
package org.springframework.vitahlin;  
  
import org.springframework.context.ApplicationContext;  
import org.springframework.context.support.ClassPathXmlApplicationContext;  
import org.springframework.vitahlin.bean.AmericaBean;  
import org.springframework.vitahlin.bean.BeijingBean;  
import org.springframework.vitahlin.bean.ChinaBean;  
  
/**  
 * 测试bean的继承  
 */  
public class ClassPathXmlApplicationContextTest1 {  
    public static void main(String[] args) {  
        ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");  
    ChinaBean chinaBean1 = (ChinaBean) context.getBean("chinaBean");  
        System.out.println(chinaBean1);  
        ChinaBean chinaBean2 = (ChinaBean) context.getBean("chinaBean");  
        System.out.println(chinaBean2);  
        BeijingBean beijingBean1 = (BeijingBean) context.getBean("beijingBean");  
        System.out.println(beijingBean1);  
        BeijingBean beijingBean2 = (BeijingBean) context.getBean("beijingBean");  
        System.out.println(beijingBean2);  
    AmericaBean americaBean1 = (AmericaBean) context.getBean("americaBean");  
        System.out.println(americaBean1);  
    AmericaBean americaBean2 = (AmericaBean) context.getBean("americaBean");  
        System.out.println(americaBean2);  
    }  
}
```


执行结果：
```shell
org.springframework.vitahlin.bean.ChinaBean@6500df86
org.springframework.vitahlin.bean.ChinaBean@6500df86
org.springframework.vitahlin.bean.BeijingBean@402a079c
org.springframework.vitahlin.bean.BeijingBean@402a079c
org.springframework.vitahlin.bean.AmericaBean@59ec2012
org.springframework.vitahlin.bean.AmericaBean@4cf777e8
```

我们通过xml来定义bean的继承关系，ChinaBean定义自身的scope属性，所以最终执行结果就是ChinaBean为单例bean，AmericaBean则不是。
这里就需要处理bean的合并，**Spring中有对应的概念  `MergedBeanDefinition`**，它并没有对应的类或接口存在，但是确实存在这个概念，并且`Spring`专门为它提供了一个生命周期回调定义接口`MergedBeanDefinitionPostProcessor`用于扩展。

跟踪 `getMergedLocalBeanDefinition`  的实现，最终实现方法是`AbstractBeanFactory#getMergedBeanDefinition` ：
```java
protected RootBeanDefinition getMergedBeanDefinition(  
      String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)  
      throws BeanDefinitionStoreException {  
  
   synchronized (this.mergedBeanDefinitions) {  
      // 1.准备一个RootBeanDefinition变量引用，用于记录要构建和最终要返回的BeanDefinition  
      RootBeanDefinition mbd = null;  
      RootBeanDefinition previous = null;  
  
      // Check with full lock now in order to enforce the same merged instance.  
      if (containingBd == null) {  
         mbd = this.mergedBeanDefinitions.get(beanName);  
      }  
  
      if (mbd == null || mbd.stale) {  
         previous = mbd;  
         // 2.判读有没有指定parent  
         if (bd.getParentName() == null) {  
            // Use copy of given root bean definition.  
            // 3.如果自己就是一个root，那么就克隆一下，生成一个新的  
            if (bd instanceof RootBeanDefinition) {  
               mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();  
            }  
            else {  
               mbd = new RootBeanDefinition(bd);  
            }  
         }  
         else {  
            // Child bean definition: needs to be merged with parent.  
            // 3. 定义父亲的BeanDefinition  
            BeanDefinition pbd;  
            try {  
               // 4.拿到这个bean的parent的名字  
               String parentBeanName = transformedBeanName(bd.getParentName());  
               // 5.当前名字不等于parentBean的名字，递归处理，一直往上找  
               if (!beanName.equals(parentBeanName)) {  
                  pbd = getMergedBeanDefinition(parentBeanName);  
               } else {  
                  BeanFactory parent = getParentBeanFactory();  
                  if (parent instanceof ConfigurableBeanFactory) {  
                     pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);  
                  } else {  
                     throw new NoSuchBeanDefinitionException(parentBeanName,  
                        "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +  
                           "': cannot be resolved without a ConfigurableBeanFactory parent");  
                  }  
               }  
            } catch (NoSuchBeanDefinitionException ex) {  
               throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,  
                  "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);  
            }  
            // Deep copy with overridden values.  
            // 6. 生成一个新的BeanDefinition，并且覆盖掉父亲的属性  
            mbd = new RootBeanDefinition(pbd);  
            mbd.overrideFrom(bd);  
         }  
  
         // Set default singleton scope, if not configured before.  
         if (!StringUtils.hasLength(mbd.getScope())) {  
            mbd.setScope(SCOPE_SINGLETON);  
         }  
  
         // A bean contained in a non-singleton bean cannot be a singleton itself.  
         // Let's correct this on the fly here, since this might be the result of         // parent-child merging for the outer bean, in which case the original inner bean         // definition will not have inherited the merged outer bean's singleton status.         if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {  
            mbd.setScope(containingBd.getScope());  
         }  
  
         // Cache the merged bean definition for the time being  
         // (it might still get re-merged later on in order to pick up metadata changes)         if (containingBd == null && isCacheBeanMetadata()) {  
	        // 7. 放到map中
            this.mergedBeanDefinitions.put(beanName, mbd);  
         }  
      }  
      if (previous != null) {  
         copyRelevantMergedBeanDefinitionCaches(previous, mbd);  
      }  
      return mbd;  
   }  
}
```

在这个方法里，就处理了`BeanDefinition` 的合并，合并后生成最终的`RootBeanDefinition` ，然后放到map里面，注意**这里的map是 `mergedBeanDefinitions` ，用来保存合并后的 `BeanDefinition` 。** 

一个`MergedBeanDefinition` 是这样一个载体:
1. 根据原始`BeanDefinition` 及其可能存在的双亲`BeanDefinition` 中的bean定义信息"合并"而得来的一个`RootBeanDefinition` ；
2. 每个Bean的创建需要的是一个MergedBeanDefinition，也就是需要基于原始BeanDefinition及其双亲BeanDefinition信息得到一个信息"合并"之后的BeanDefinition；
3. Spring框架同时提供了一个机会给框架其他部分，或者开发人员用于在bean创建过程中，MergedBeanDefinition生成之后，bean属性填充之前，对该bean和该MergedBeanDefinition做一次回调，相应的回调接口是MergedBeanDefinitionPostProcessor。
4. MergedBeanDefinition没有相应的Spring建模，它是处于一个内部使用目的合并自其它BeanDefinition对象，其具体对象所使用的实现类类型是RootBeanDefinition。