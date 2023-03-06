---
title: Spring源码10-Bean销毁
<!-- description: 这是一个副标题 -->
date: 2023-03-05
lastmod: 2023-03-05
slug: 01GTVCEZZJF24WKGKFHF4DS7A1
categories:
    - Spring
tags:
    - Spring
---


## 前言

在前文中，我们讲到了 Bean 实例化流程的最后，会注册 Bean 的销毁逻辑，本文就介绍下 Bean 的销毁流程及其源码。

## registerDisposableBeanIfNecessary 源码

回顾下 Bean 的实例化流程，创建 Bean 完成后并不是就直接结束了，我们有时候还需要定义 Bean 销毁的逻辑 `AbstractAutowireCapableBeanFactory#doCreateBean` :
```java
try {  
    registerDisposableBeanIfNecessary(beanName, bean, mbd);  
} catch (BeanDefinitionValidationException ex) {  
    throw new BeanCreationException(  
        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
}
```

`registerDisposableBeanIfNecessary`  方法就是用来定义 Bean 销毁的，跟进方法源码 ` AbstractBeanFactory #registerDisposableBeanIfNecessary` ：
```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {  
    AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);  
    if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {  
        if (mbd.isSingleton()) {  
            registerDisposableBean(beanName, new DisposableBeanAdapter(  
                bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));  
        } else {  
            // A bean with a custom scope...  
            Scope scope = this.scopes.get(mbd.getScope());  
            if (scope == null) {  
                throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");  
            }  
            scope.registerDestructionCallback(beanName, new DisposableBeanAdapter(  
                bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));  
        }  
    }  
}
```

方法会先判断是不是 Bean 的作用域是不是 Prototype 类类型，`requiresDestruction`  方法判断 Bean 是否有定义销毁逻辑，跟进方法 ` AbstractBeanFactory#requiresDestruction ` ：
```java
protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) {    
    return (bean.getClass() != NullBean.class && (DisposableBeanAdapter.hasDestroyMethod(bean, mbd) ||  
        (hasDestructionAwareBeanPostProcessors() && DisposableBeanAdapter.hasApplicableProcessors(  
            bean, getBeanPostProcessorCache().destructionAware))));  
}
```

可以看到 `requiresDestruction` 方法只要满足下面任意一个即可：
- HasDestroyMethod 检查给定 bean 是否有任何类型的 destroy 销毁的方法可调用；
- HasApplicableProcessors 检查是否有 DestructionAwareBeanPostProcessor 的实现类，并且 requiresDestruction 方法返回 true。

在满足了上述条件之后，如果 `bean` 的作用域是单例的时，创建 `DisposableBeanAdapter` 进行注册（添加到 `disposableBeans` 集合 `LinkedHashMap` 中），来看方法 `DefaultSingletonBeanRegistry#registerDisposableBean` ：
```java
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

public void registerDisposableBean(String beanName, DisposableBean bean) {  
    synchronized (this.disposableBeans) {  
        this.disposableBeans.put(beanName, bean);  
    }  
}
```

到这里，Bean 实例化时对 Bean 销毁的处理流程就完成了，最终就是注册一个 `DisposableBeanAdapter` 对象。

## Spring 关闭时的处理

Bean 销毁是发送在 Spring 容器关闭过程中的，所以我们同样需要来看 Spring 的销毁流程。

### registerShutdownHook

Spring 会先去调用注册的关闭钩子 `ShutdownHook`，通过这个方式进行销毁前的动作处理 `AbstractApplicationContext#registerShutdownHook` ：
```java
@Override  
public void registerShutdownHook() {  
    /*** 容器是否有注册对应的关闭钩子 */  
    if (this.shutdownHook == null) {  
        // No shutdown hook registered yet.  
        this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {  
            @Override  
            public void run() {  
                synchronized (startupShutdownMonitor) {  
                    doClose();  
                }  
            }  
        };  
        Runtime.getRuntime().addShutdownHook(this.shutdownHook);  
    }  
}
```

继续跟进 `doClose`  方法：
```java
protected void doClose() {  
    if (this.active.get() && this.closed.compareAndSet(false, true)) {  
        if (logger.isDebugEnabled()) {  
            logger.debug("Closing " + this);  
        }  
        if (!NativeDetector.inNativeImage()) {  
            LiveBeansView.unregisterApplicationContext(this);  
        }  
        try {  
            publishEvent(new ContextClosedEvent(this));  
        } catch (Throwable ex) {  
            logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);  
        }  
        if (this.lifecycleProcessor != null) {  
            try {  
                this.lifecycleProcessor.onClose();  
            } catch (Throwable ex) {  
                logger.warn("Exception thrown from LifecycleProcessor on context close", ex);  
            }  
        }  
        destroyBeans();  
        closeBeanFactory();  
        onClose();  
        if (this.earlyApplicationListeners != null) {  
            this.applicationListeners.clear();  
            this.applicationListeners.addAll(this.earlyApplicationListeners);  
        }  
        this.active.set(false);  
    }  
}
```
其中，destroyBean 就是 Bean 销毁的逻辑。

### destroySingletons

最终执行的方法是 `DefaultSingletonBeanRegistry#destroySingletons` ：
```java
public void destroySingletons() {  
    if (logger.isTraceEnabled()) {  
        logger.trace("Destroying singletons in " + this);  
    }  
    synchronized (this.singletonObjects) {  
        this.singletonsCurrentlyInDestruction = true;  
    }  
    String[] disposableBeanNames;  
    synchronized (this.disposableBeans) {  
        disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());  
    }  
    for (int i = disposableBeanNames.length - 1; i >= 0; i--) {  
        destroySingleton(disposableBeanNames[i]);  
    }  
    this.containedBeanMap.clear();  
    this.dependentBeanMap.clear();  
    this.dependenciesForBeanMap.clear();  
    clearSingletonCache();  
}
```

`DefaultSingletonBeanRegistry#destroySingleton` ：
```java
public void destroySingleton(String beanName) {  
    // Remove a registered singleton of the given name, if any.  
    removeSingleton(beanName);  
    // Destroy the corresponding DisposableBean instance.  
    DisposableBean disposableBean;  
    synchronized (this.disposableBeans) {  
        disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);  
    }  
    destroyBean(beanName, disposableBean);  
}
```

最终调用的方法是 `DefaultSingletonBeanRegistry#destroyBean` ：
```java
protected void destroyBean(String beanName, @Nullable DisposableBean bean) {  
    // Trigger destruction of dependent beans first...  
    Set<String> dependencies;  
    synchronized (this.dependentBeanMap) {  
        // Within full synchronization in order to guarantee a disconnected Set  
        dependencies = this.dependentBeanMap.remove(beanName);  
    }  
    if (dependencies != null) {  
        if (logger.isTraceEnabled()) {  
            logger.trace("Retrieved dependent beans for bean '" + beanName + "': " + dependencies);  
        }  
        for (String dependentBeanName : dependencies) {  
            destroySingleton(dependentBeanName);  
        }  
    }  
    // Actually destroy the bean now...  
    if (bean != null) {  
        try {  
            bean.destroy();  
        } catch (Throwable ex) {  
            if (logger.isWarnEnabled()) {  
                logger.warn("Destruction of bean with name '" + beanName + "' threw an exception", ex);  
            }  
        }  
    }  
    // Trigger destruction of contained beans...  
    Set<String> containedBeans;  
    synchronized (this.containedBeanMap) {  
        // Within full synchronization in order to guarantee a disconnected Set  
        containedBeans = this.containedBeanMap.remove(beanName);  
    }  
    if (containedBeans != null) {  
        for (String containedBeanName : containedBeans) {  
            destroySingleton(containedBeanName);  
        }  
    }  
    // Remove destroyed bean from other beans' dependencies.  
    synchronized (this.dependentBeanMap) {  
        for (Iterator<Map.Entry<String, Set<String>>> it = this.dependentBeanMap.entrySet().iterator(); it.hasNext(); ) {  
            Map.Entry<String, Set<String>> entry = it.next();  
            Set<String> dependenciesToClean = entry.getValue();  
            dependenciesToClean.remove(beanName);  
            if (dependenciesToClean.isEmpty()) {  
                it.remove();  
            }  
        }  
    }  
    // Remove destroyed bean's prepared dependency information.  
    this.dependenciesForBeanMap.remove(beanName);  
}
```

这里，先取出 Bean 的依赖方，先销毁依赖方（就是之前 Bean  装配提到的 ` @DependsOn ` 标志的依赖方），然后再调用自身的 `destroy`  方法。

### bean.destroy

`bean.destroy()` 最终调用的是 `DisposableBeanAdapter#destroy`：
```java
@Override  
public void destroy() {  
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {  
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {  
            processor.postProcessBeforeDestruction(this.bean, this.beanName);  
        }  
    }  
    if (this.invokeDisposableBean) {  
        if (logger.isTraceEnabled()) {  
            logger.trace("Invoking destroy() on bean with name '" + this.beanName + "'");  
        }  
        try {  
            if (System.getSecurityManager() != null) {  
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {  
                    ((DisposableBean) this.bean).destroy();  
                    return null;                
                }, this.acc);  
            } else {  
                ((DisposableBean) this.bean).destroy();  
            }  
        } catch (Throwable ex) {  
            String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";  
            if (logger.isDebugEnabled()) {  
                logger.warn(msg, ex);  
            } else {  
                logger.warn(msg + ": " + ex);  
            }  
        }  
    }  
    if (this.destroyMethod != null) {  
        invokeCustomDestroyMethod(this.destroyMethod);  
    } else if (this.destroyMethodName != null) {  
        Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);  
        if (methodToInvoke != null) {  
            invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));  
        }  
    }  
}
```

包括以下几项：
1. `processor.postProcessBeforeDestruction(this.bean, this.beanName)` 调用 DestructionAwareBeanPostProcessor 来执行销毁方法；
2.  `((DisposableBean) this.bean).destroy()`  执行 ` DisposableBean ` 的 ` destroy ` 方法；
3.  `invokeCustomDestroyMethod(this.destroyMethod)`  执行 ` init-method ` 指定的 ` destroy ` 方法；
4.  `invokeCustomDestroyMethod` 执行 ` @Bean ` 属性指定的 ` destroy ` 方法。

## Bean 销毁流程

我们平常定义 Bean 销毁的方式有以下几种：
- 注解标注 @preDestory 标注方法
- 实现 DisposableBean 接口的 destroy()方法
- 自定义销毁方法
    - xml 配置
    - Java 注解
    - Java API

可以总结 Spring 中 Bean 销毁的流程，在 Spring 容器关闭过程时：
- 取出需要执行销毁方法的 `bean`（前面添加到了 `disposableBeans` 中）；
- 遍历每个 bean 执行销毁方法；
- 取出 bean 的依赖方，先销毁依赖方（就是之前 bean 装配提到的 @DependsOn 标志的依赖方）；
- 执行 bean 的 destroy 销毁方法，包括下面 4 项
    - 调用 DestructionAwareBeanPostProcessor 来执行销毁方法；
    - 执行 DisposableBean 的 destroy 方法；
    - 执行 init-method 指定的 destroy 方法；
    - 执行 @Bean 属性指定的 destroy 方法；
- 销毁内部类对象；
- 从依赖关系中去掉自己。

所以这里，Bean 销毁的执行顺序如下：
- 注解标注> 接口实现 > 自定义，@PreDestroy > DisposableBean > destroyMethod

这里涉及到一个设计模式：**适配器模式**。

在销毁时，Spring 会找出实现了 DisposableBean 接口的 Bean。
但是我们在定义一个 Bean 时，如果这个 Bean 实现了 DisposableBean 接口，或者实现了 AutoCloseable 接口，或者在 BeanDefinition 中指定了 destroyMethodName，那么这个 Bean 都属于“DisposableBean”，这些 Bean 在容器关闭时都要调用相应的销毁方法。

所以，这里就需要进行适配，将实现了 DisposableBean 接口、或者 AutoCloseable 接口等适配成实现了 DisposableBean 接口，所以就用到了 DisposableBeanAdapter，会把实现了 AutoCloseable 接口的类封装成 DisposableBeanAdapter，而 DisposableBeanAdapter 实现了 DisposableBean 接口。
