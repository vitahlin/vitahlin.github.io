---
title: Spring源码8-Bean实例化之getBean
<!-- description: 这是一个副标题 -->
date: 2023-03-02
lastmod: 2023-03-02
slug: 01GTV8WQYVKNDNNYZ4T0JDSZQZ
categories:
    - Spring
tags:
    - Spring
---

## 前言

前文中我们已经大致说明了 Bean 的实例化流程，但是具体如何实例化还并未介绍，本文就详细介绍 Spring 的 getBean 方法，通过分析 getBean 方法的源码来了解 Bean 实例化的详细流程。

## doGetBean方法

首先，来看 getBean 方法的源码 `AbstractBeanFactory#getBean` ：
```java
public Object getBean(String name) throws BeansException {  
    return doGetBean(name, null, null, false);  
}
```

可以看到，这里是直接调用了方法 `doGetBean`，`doGetBean`  方法源码如下：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210141643939.png)

大致分为5个部分：
1. transformedBeanName 对名称进行转换
2. getSingleton 从单例池容器缓存中获取bean
3. if代码块，从缓存中获取到了Bean进行后续的操作
4. else代码块，缓存中没有则进行Bean的创建
5. adaptBeanInstance 进行类型转换

### transformedBeanName-处理Bean的名称

这一步是处理 Bean 的名称，传进来一个 BeanName 有可能是别名，也有可能是 FactoryBean 的名称。在这里统一处理 BeanName，跟进方法 `AbstractBeanFactory#transformedBeanName` ：

```java
protected String transformedBeanName(String name) {  
    /**  
     * 先调用transformedBeanName去除前缀，  
     * 再调用canonicalName获取别名  
     */  
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));  
}
```

这里是调用 `transformedBeanName` 处理 FactoryBean 的情况，再调用 `canonicalName`  方法处理别名。

先来看 `BeanFactoryUtils#transformedBeanName` ：
```java
public static String transformedBeanName(String name) {  
    Assert.notNull(name, "'name' must not be null");  
    /*** FACTORY_BEAN_PREFIX = "&"，不包含前缀直接返回 */  
    if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {  
        return name;  
    }  
    return transformedBeanNameCache.computeIfAbsent(name, beanName -> {  
        /*** 循环处理，带有多个&的都会去掉 */  
        do {  
            beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());  
        }  
        while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));  
        return beanName;  
    });  
}
```
判断是否是 `FactoryBean`  名称的格式，如果不是直接返回，如果是则处理前缀。

再看方法 `SimpleAliasRegistry#canonicalName` ：
```java
public String canonicalName(String name) {  
    String canonicalName = name;  
    /**  
     * Handle aliasing...     
     * 处理别名  
     */  
    String resolvedName;  
    do {  
        /*** 通过aliasMap获取BeanName，直到不再有别名 */  
        resolvedName = this.aliasMap.get(canonicalName);  
        /*** 可能存在 A->B->C->D, 所以需要一直循环到没有别名为止 */  
        if (resolvedName != null) {  
            canonicalName = resolvedName;  
        }  
    }  
    while (resolvedName != null);  
    return canonicalName;  
}
```

这里确定最终的名称，一个 Bean 可能会定义很多别名，这个时候需要确定最根本的那个 name，用这个最根本的 name 来作为 BeanName 去进行后续的操作。这里同样有个循环去处理，因为别名也会有多重，会存在别名的别名这种情况。

### getSingleton-从单例池中获取单例Bean

在上一步中我们已经获取到了真正的 BeanName，那么接下来，就可以利用这个 `BeanName`  到容器的缓存中尝试获取 Bean。

如果之前已经创建过，这里就可以直接获取到 Bean。这里的缓存包括三级，但是**这三级缓存并不是包含的关系，而是一种互斥的关系**，一个 Bean 无论处于何种状态，它在同一时刻只能处于某个缓存当中。这个缓存是 Spring 中用来处理循环依赖的，这里我们就简单理解为，**当前就是根据 BeanName 从单例池中获取对应的单例 Bean，详细的源码后续在讲解 Spring 循环依赖的处理中介绍。**

**这里即使它是原型Bean，也会调用这个方法。** Spring 中默认大部分 Bean 为单例 Bean。

### 缓存池中已经存在Bean的处理

如果直接从单例池中获取到了这个 bean(sharedInstance)，我们能直接返回吗？  
当然不能，因为获取到的Bean可能是一个 FactoryBean。如果我们传入的 name 是 &+beanName 这种形式的话，那是可以返回的。
但是我们传入的更可能是一个 beanName，那么这个时候Spring就还需要调用这个 sharedInstance 的getObject 方法来创建真正被需要的Bean。

所以这里需要做判断是否是 FactoryBean：
```java
if (sharedInstance != null && args == null) {  
    if (logger.isTraceEnabled()) {  
        if (isSingletonCurrentlyInCreation(beanName)) {  
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +  
                "' that is not fully initialized yet - a consequence of a circular reference");  
        } else {  
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");  
        }  
    }  
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);  
}
```

if 语句进去，最终执行的方法是 `getObjectForBeanInstance` ，跟进去可以看到这里会进行一些类型的判断，会尝试从缓存获取，最后会调用 `getObjectFromFactoryBean` 方法从 `FactoryBean` 里获取实例对象。

`AbstractBeanFactory#getObjectForBeanInstance` ：
```java
protected Object getObjectForBeanInstance(  
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {  
    /**  
     * 如果指定的name是工厂相关（以&为前缀）且 beanInstance又不是FactoryBean类型则验证不通过  
     */  
    if (BeanFactoryUtils.isFactoryDereference(name)) {  
        if (beanInstance instanceof NullBean) {  
            return beanInstance;  
        }  
        if (!(beanInstance instanceof FactoryBean)) {  
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());  
        }  
        if (mbd != null) {  
            mbd.isFactoryBean = true;  
        }  
        return beanInstance;  
    }  
    /**  
     * 现在我们有了 bean 实例，它可能是普通的 bean 或 FactoryBean。  
     * 如果它是一个 FactoryBean，我们使用它来创建一个 bean 实例，除非调用者实际上想要一个对工厂的引用。  
     * 如果是普通bean，直接返回了  
     */  
    if (!(beanInstance instanceof FactoryBean)) {  
        return beanInstance;  
    }  
    /*** 加载factoryBean */  
    Object object = null;  
    if (mbd != null) {  
        mbd.isFactoryBean = true;  
    } else {  
        /*** 尝试从缓存中获取 */  
        object = getCachedObjectForFactoryBean(beanName);  
    }  
    if (object == null) {  
        /**  
         * 到这里可以确定 beanInstance 一定是 FactoryBean 类型  
         */  
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;  
        /**  
         * Caches  object obtained from FactoryBean if it is a singleton.         
         * 如果是单例，则缓存从 FactoryBean 获得的对象。  
         * containsBeanDefinition 检测 beanDefinitionMap 中也就是在所有已经加载的类中检测是否定义 beanName  
         */        
         if (mbd == null && containsBeanDefinition(beanName)) {  
            /**  
             * 将存储XML文件的 GenericBeanDefinition 转换为 RootBeanDefinition，  
             * 如果指定的 beanName 是子 bean 的话会同时合并父类的相关属性  
             */  
            mbd = getMergedLocalBeanDefinition(beanName);  
        }  
        /*** 是否是用用户自己定义的还是用程序本身定义的 */  
        boolean synthetic = (mbd != null && mbd.isSynthetic());  
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);  
    }  
    return object;  
}
```

`FactoryBeanRegistrySupport#getObjectFromFactoryBean` ：
```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {  
    /*** 如果是单例 */  
    if (factory.isSingleton() && containsSingleton(beanName)) {  
        /*** 加锁，保证单例 */  
        synchronized (getSingletonMutex()) {  
            /*** 先从缓存中获取 */  
            Object object = this.factoryBeanObjectCache.get(beanName);  
            if (object == null) {  
                /*** 从factoryBean中获取Bean */  
                object = doGetObjectFromFactoryBean(factory, beanName);  
                // Only post-process and store if not put there already during getObject() call above  
                // (e.g. because of circular reference processing triggered by custom getBean calls)                
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);  
                if (alreadyThere != null) {  
                    object = alreadyThere;  
                } else {  
                    if (shouldPostProcess) {  
                        if (isSingletonCurrentlyInCreation(beanName)) {  
                            // Temporarily return non-post-processed object, not storing it yet..  
                            return object;  
                        }  
                        beforeSingletonCreation(beanName);  
                        try {  
                            /*** 调用ObjectFactory的后置处理器 */  
                            object = postProcessObjectFromFactoryBean(object, beanName);  
                        } catch (Throwable ex) {  
                            throw new BeanCreationException(beanName,  
                                "Post-processing of FactoryBean's singleton object failed", ex);  
                        } finally {  
                            afterSingletonCreation(beanName);  
                        }  
                    }  
                    if (containsSingleton(beanName)) {  
                        this.factoryBeanObjectCache.put(beanName, object);  
                    }  
                }  
            }  
            return object;  
        }  
    } else {  
        /*** 从FactoryBean获取对象 */  
        Object object = doGetObjectFromFactoryBean(factory, beanName);  
        if (shouldPostProcess) {  
            try {  
                /*** 后置处理从 FactoryBean 获取的对象 */  
                object = postProcessObjectFromFactoryBean(object, beanName);  
            } catch (Throwable ex) {  
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);  
            }  
        }  
        return object;  
    }  
}
```

这里的主要逻辑就是根据 BeanName 从缓存池中获取缓存的 Bean
- 如果 Bean 是单例 Bean 并且已经存在，则直接返回；
- 如果 Bean 是 FactoryBean，那就调用 FactoryBean 生成对象。


### 真正进入创建Bean的流程

经过前面这么多铺垫，才真正走到了创建 Bean 的地方，即 else 的代码块。这里会比较复杂且啰嗦，需要点耐心看完。

```java
if (isPrototypeCurrentlyInCreation(beanName)) {  
    throw new BeanCurrentlyInCreationException(beanName);  
}
```
一开始，先对原型类型的循环依赖进行校验，原型 Bean 出现循环依赖直接抛异常，中断流程，不继续往下走。


```java
BeanFactory parentBeanFactory = getParentBeanFactory();  
 if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {  
    String nameToLookup = originalBeanName(name);  
    if (parentBeanFactory instanceof AbstractBeanFactory) {  
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(  
            nameToLookup, requiredType, args, typeCheckOnly);  
    } else if (args != null) {  
        // Delegation to parent with explicit args.  
        return (T) parentBeanFactory.getBean(nameToLookup, args);  
    } else if (requiredType != null) {  
        // No args -> delegate to standard getBean method.  
        return parentBeanFactory.getBean(nameToLookup, requiredType);  
    } else {  
        return (T) parentBeanFactory.getBean(nameToLookup);  
    }  
}
```
这部分代码，主要处理的是存在父 BeanFactory，并且当前的 BeanFactory 里面没有这个 BeanDefinition，就尝试从父 BeanFactory 中获取，如果有对应 Bean 对象就直接返回了。


```java
if (!typeCheckOnly) {  
    markBeanAsCreated(beanName);  
}
```
因为接下来，我们要开始创建 Bean 了，所以这里先把这个 Bean 标记为正在创建中，跟进方法`markBeanAsCreated` ，`AbstractBeanFactory#markBeanAsCreated` ：
```java
protected void markBeanAsCreated(String beanName) {  
    if (!this.alreadyCreated.contains(beanName)) {  
        synchronized (this.mergedBeanDefinitions) {  
            if (!this.alreadyCreated.contains(beanName)) {  
                // Let the bean definition get re-merged now that we're actually creating  
                // the bean... just in case some of its metadata changed in the meantime.                clearMergedBeanDefinition(beanName);  
                this.alreadyCreated.add(beanName);  
            }  
        }  
    }  
}
```
其实就是将当前这个 beanName 放入到  `alreadyCreated`  这个集合中。


接着，处理被 `@DependsOn` 注解标注的依赖，比如 userService 依赖 orderService，那么在这里就会去创建 orderService ：
```java
String[] dependsOn = mbd.getDependsOn();  
if (dependsOn != null) {  
    /*** 遍历所有声明的依赖 */  
    for (String dep : dependsOn) {  
        /**  
         * 判断beanName是不是被dep依赖了，如果是，则出现了循环依赖，抛出异常。  
         * spring无法解决由dependsOn注解引出的循环依赖  
         */  
        if (isDependent(beanName, dep)) {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");  
        }  
        /*** 注册bean跟其依赖的依赖关系，key为依赖，value为依赖所从属的bean */  
        registerDependentBean(dep, beanName);  
        try {  
            /**  
             * 在这里，调用getBean就又执行了当前方法，只不过参数不一样，  
             * 会去创建依赖的bean，如果创建失败则抛出异常  
             */            
             getBean(dep);  
        } catch (NoSuchBeanDefinitionException ex) {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);  
        }  
    }  
}
```


创建依赖的Bean对象后，我们就可以创建当前所需要的 Bean 对象。这里又对 Bean 的类型进行判断，区分三种不同的创建逻辑：
1. 单例Bean
2. 原型Bean
3. 既不是单例Bean也不是原型Bean的其他作用域的Bean

#### 原型Bean的创建

先来看原型Bean的创建：
```java
else if (mbd.isPrototype()) {  
    Object prototypeInstance = null;  
    try {  
        beforePrototypeCreation(beanName);  
        prototypeInstance = createBean(beanName, mbd, args);  
    } finally {  
        afterPrototypeCreation(beanName);  
    }  
    beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);  
}
```

原型Bean的创建流程比较简单：
1. `beforePrototypeCreation` 方法先标记下当前的 Bean 正在创建中；
2. 调用 `createBean(beanName, mbd, args)` 进行创建Bean，createBean 的流程后续详细分析，这里直接理解为就是创建一个 Bean；
3. afterPrototypeCreation 去掉表示正在创建的标记；
4. getObjectForBeanInstance，从 beanInstannce 中获取公开的 Bean 对象，主要处理 beanInstance 是 FactoryBean 对象的情况，如果不是 FactoryBean 会直接返回 beanInstance 实例。

#### 既不是单例 Bean 也不是原型 Bean的处理

```java
 else {  
    String scopeName = mbd.getScope();  
    if (!StringUtils.hasLength(scopeName)) {  
        throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");  
    }  
    Scope scope = this.scopes.get(scopeName);  
    if (scope == null) {  
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");  
    }  
    try {  
        Object scopedInstance = scope.get(beanName, () -> {  
            beforePrototypeCreation(beanName);  
            try {  
                return createBean(beanName, mbd, args);  
            } finally {  
                afterPrototypeCreation(beanName);  
            }  
        });  
        beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);  
    } catch (IllegalStateException ex) {  
        throw new ScopeNotActiveException(beanName, scopeName, ex);  
    }  
}
```
这里先拿到 Scope 的实现类，然后这里是一段 lambda 代码，实际上调用的 Scope 实现类的 `get`  方法。

我们以 `SessionScope`  为例，跟进 `SessionScope#get`  方法：
```java
@Override  
public Object get(String name, ObjectFactory<?> objectFactory) {  
   Object mutex = RequestContextHolder.currentRequestAttributes().getSessionMutex();  
   synchronized (mutex) {  
      return super.get(name, objectFactory);  
   }
}
```
参数中 name 是 Bean 的名字，**objectFactory 就是创建 Bean 的 lambda 表达式** 。

跟进 `super.get`  方法 `AbstractRequestAttributesScope#get` :
```java
@Override  
public Object get(String name, ObjectFactory<?> objectFactory) {  
    RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();  
    /**  
     * name是bean的名字，这里有一个getAttribute方法，  
     * 看它的实现ServletRequestAttributes  
     */    
    Object scopedObject = attributes.getAttribute(name, getScope());  
    if (scopedObject == null) {  
        /**  
         * 如果没有拿到，执行传进来的第二个参数lambda表达式，创建一个bean对象  
         */  
        scopedObject = objectFactory.getObject();  
        /*** 创建完成后，再set进去 */  
        attributes.setAttribute(name, scopedObject, getScope());  
        // Retrieve object again, registering it for implicit session attribute updates.  
        // As a bonus, we also allow for potential decoration at the getAttribute level.        
        Object retrievedObject = attributes.getAttribute(name, getScope());  
        if (retrievedObject != null) {  
            // Only proceed with retrieved object if still present (the expected case).  
            // If it disappeared concurrently, we return our locally created instance.            
            scopedObject = retrievedObject;  
        }  
    }  
    return scopedObject;  
}
```

继续跟进，最终执行的方法是 `ServletRequestAttributes#getAttribute` ：
```java
@Override  
public Object getAttribute(String name, int scope) {  
    if (scope == SCOPE_REQUEST) {  
        /*** 如果是request，就直接调用getAttribute */  
        if (!isRequestActive()) {  
            throw new IllegalStateException(  
                "Cannot ask for request attribute - request is not active anymore!");  
        }  
        return this.request.getAttribute(name);  
    } else {  
        /**  
         * 如果是session，先调用session，再调用getAttribute  
         */        
        HttpSession session = getSession(false);  
        if (session != null) {  
            try {  
                Object value = session.getAttribute(name);  
                if (value != null) {  
                    this.sessionAttributesToUpdate.put(name, value);  
                }  
                return value;  
            } catch (IllegalStateException ex) {  
                // Session invalidated - shouldn't usually happen.  
            }  
        }  
        return null;  
    }  
}
```

到这里，我们就能串联起整个流程。如果是 SessionScope，同一 session 中的 Bean 都是一样的，Bean 先从 HttpSession 中获取，如果不存在则同样执行 createBean 进行创建，创建完成后把 Bean 放到集合中去。


#### 单例Bean的创建

分析完原型 Bean 和其他作用域类型 Bean 的创建流程，就到了最重要的单例 Bean 创建的环节，这也是创建 Bean 的核心代码：
```java
if (mbd.isSingleton()) {  
    sharedInstance = getSingleton(beanName, () -> {  
        try {  
            return createBean(beanName, mbd, args);  
        } catch (BeansException ex) {  
            destroySingleton(beanName);  
            throw ex;  
        }  
    });  
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);  
}
```

#### getSingleton

这是一段 lambda 表达式，我们先来看 getSingleton 方法：
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {  
    Assert.notNull(beanName, "Bean name must not be null");  
    synchronized (this.singletonObjects) {  
        Object singletonObject = this.singletonObjects.get(beanName);  
        if (singletonObject == null) {  
            if (this.singletonsCurrentlyInDestruction) {  
                throw new BeanCreationNotAllowedException(beanName,  
                    "Singleton bean creation not allowed while singletons of this factory are in destruction " +  
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");  
            }  
            if (logger.isDebugEnabled()) {  
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");  
            }  
            beforeSingletonCreation(beanName);  
            boolean newSingleton = false;  
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);  
            if (recordSuppressedExceptions) {  
                this.suppressedExceptions = new LinkedHashSet<>();  
            }  
            try {  
                singletonObject = singletonFactory.getObject();  
                newSingleton = true;  
            } catch (IllegalStateException ex) {  
                singletonObject = this.singletonObjects.get(beanName);  
                if (singletonObject == null) {  
                    throw ex;  
                }  
            } catch (BeanCreationException ex) {  
                if (recordSuppressedExceptions) {  
                    for (Exception suppressedException : this.suppressedExceptions) {  
                        ex.addRelatedCause(suppressedException);  
                    }  
                }  
                throw ex;  
            } finally {  
                if (recordSuppressedExceptions) {  
                    this.suppressedExceptions = null;  
                }  
                afterSingletonCreation(beanName);  
            }  
            if (newSingleton) {  
                addSingleton(beanName, singletonObject);  
            }  
        }  
        return singletonObject;  
    }  
}
```

1. 这个方法的流程并不复杂，首先 `Object singletonObject = this.singletonObjects.get(beanName)`  从单例池中获取 Bean 对象，如果不为空则直接返回，为空表示 Bean 还未创建，继续执行创建流程。
2.  `beforeSingletonCreation(beanName)` 则是在 Bean 创建前先打个标记，表示当前这个 Bean 正在创建。
3. `singletonObject = singletonFactory.getObject()`  ，`getSingleton`  这个方法的第二个参数 singletonFactory 实际上是一个 lambda 表达式，所以这里就会去执行 lambda 表达式，执行完创建一个对象。
4. `afterSingletonCreation(beanName)` 在单例完成创建后，将 beanName 从 singletonsCurrentlyInCreation 中移除，标志着这个单例已经完成了创建。
5. 创建完成后，就通过 `addSingleton(beanName, singletonObject)` 加入到单例池中去，下次直接获取即可。

我们可以来看下 addSingleton 源码 DefaultSingletonBeanRegistry.addSingleton：
```java
protected void addSingleton(String beanName, Object singletonObject) {  
    synchronized (this.singletonObjects) {  
        this.singletonObjects.put(beanName, singletonObject);  
        this.singletonFactories.remove(beanName);  
        this.earlySingletonObjects.remove(beanName);  
        this.registeredSingletons.add(beanName);  
    }  
}
```
就是把创建出来的 Bean 对象放到缓存池 singletonObjects 中去。

这就是 Bean 的创建流程，步骤 3 中的 lambda 其实是调用了 createBean 的方法，后续我们将继续分析 createBean 的源码。

## 结语

本文分析了 Bean 实例化流程中的 doGetBean 方法，介绍了单例 Bean 、原型 Bean，以及通过 Scope 作用域为例介绍其他作用域 Bean 的创建流程，最终 Bean 的创建都是通过 lambda 方式调用了 createBean 方法，后续将进一步介绍 createBean 的流程。