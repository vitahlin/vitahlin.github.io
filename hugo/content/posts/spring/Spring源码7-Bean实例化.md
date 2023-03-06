---
title: Spring源码6-Bean实例化
<!-- description: 这是一个副标题 -->
date: 2023-03-04
lastmod: 2023-03-04
slug: 01GTV8XA5NDA8R6XQDT0XFSQYT
categories:
    - Spring
tags:
    - Spring
---


## 前言

先回顾下Bean实例化的真正入口，跟进源码 `DefaultListableBeanFactory#preInstantiateSingletons` 来看实例化的流程：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102210871.png)

这里主要有3部分逻辑：
1. BeanDefnition 的合并-getMergedLocalBeanDefinition；
2. BeanDefnition 的实例化-getBean；
3. SmartInitializingSingleton 接口的实现。

前文中我们已经得到一个合并后的 BeanDefinition，那么接下来就是根据合并后的 BeanDefinition 进行实例化。

## getBean-Bean的实例化

获取 `RootBeanDefinition` 完成后，接下来就调用 `getBean(beanName)` 进行 Bean 的实例化了，源码部分如下：
```java
if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {  
    /*** 判断是不是FactoryBean */  
    if (isFactoryBean(beanName)) {  
        /**  
         * 如果是一个factoryBean的话，先创建这个factoryBean  
         * 创建factoryBean时，需要在beanName前面拼接一个&符号  
         */  
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
        if (bean instanceof FactoryBean) {  
            FactoryBean<?> factory = (FactoryBean<?>) bean;  
            boolean isEagerInit;  
            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {  
                isEagerInit = AccessController.doPrivileged(  
                    (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,  
                    getAccessControlContext());  
            } else {  
                /**  
                 * 判断是否是一个SmartFactoryBean，并且不是懒加载的，就意味着,  
                 * 在创建了这个factoryBean之后要立马调用它的getObject方法创建另外一个Bean  
                 */                
                 isEagerInit = (factory instanceof SmartFactoryBean &&  
                    ((SmartFactoryBean<?>) factory).isEagerInit());  
            }  
            if (isEagerInit) {  
                getBean(beanName);  
            }  
        }  
    } else {  
        /*** 不是factoryBean的话，直接创建就行了 */  
        getBean(beanName);  
    }  
}
```

当然这里的创建还是分了**工厂Bean**和**非工厂Bean**两个逻辑
- 如果不是 FactoryBean 的话，那么很简单直接调用 `getBean` 方法 获取实例化后的 Bean 对象
- 如果是一个工厂 Bean，那么 `getBean(beanName)` 这一步只会创建一个工厂 Bean，接下来会通过 `isEagerInit` 参数判断是否需要初始化工厂 Bean 的对象，如果需要，再调用 `getBean(beanName)` 去立马获取工厂 Bean 需要生产的对象。我们可以新建测试类来验证。

### 判断是否是FactoryBean

来看 `AbstractBeanFactory#isFactoryBean` 方法，这里传入的参数是字符串类型的 baneName：
```java
@Override  
public boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException {  
    String beanName = transformedBeanName(name);  
    Object beanInstance = getSingleton(beanName, false);  
    if (beanInstance != null) {  
        return (beanInstance instanceof FactoryBean);  
    }  
     if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {  
        return ((ConfigurableBeanFactory) getParentBeanFactory()).isFactoryBean(name);  
    }  
    return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));  
}
```

首先，`Object beanInstance = getSingleton(beanName, false)` 直接从单例池中获取对应的 Bean 对象，看是否已经存在，如果已经存在的话判断是不是 FactoryBean。

如果不存在单例对象，只能通过 BeanDefinition 对象来判断了，`containsBeanDefinition(beanName)` 表示容器是否存在该 BeanDefinition 对象，`getParentBeanFactory()` 则是获取父 BeanFactory。

继续看 `isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName))` 的逻辑，`AbstractBeanFactory#isFactoryBean` ：
```java
protected boolean isFactoryBean(String beanName, RootBeanDefinition mbd) {  
    Boolean result = mbd.isFactoryBean;  
    if (result == null) {  
        Class<?> beanType = predictBeanType(beanName, mbd, FactoryBean.class);  
        /** 判断是不是FactoryBean接口 */  
        result = (beanType != null && FactoryBean.class.isAssignableFrom(beanType));  
        mbd.isFactoryBean = result;  
    }  
    return result;  
}
```
`predictBeanType` 是根据 BeanDefinition 去拿 `beanClass` 的属性，就是拿到当前 BeanDefinition 对应的类，然后去判断是不是 FactoryBean 接口。 

到这里就完成了 FactoryBean 的判断，接下来就来看当前 Bean 如果是FactoryBean 那么应该怎么处理。

### 当前是FactoryBean

新建 FactoryBean 测试类：
```java
package org.springframework.vitahlin.bean.factory;  
  
public class Student {  
    private String name;  
    private String code;  
    public Student(String name, String code) {  
        this.name = name;  
        this.code = code;  
    }  
    public Student() {  
    }    @Override  
    public String toString() {  
        return "Student{}";  
    }  
}
```

```java
package org.springframework.vitahlin.bean.factory;  
  
import org.springframework.beans.factory.FactoryBean;  
import org.springframework.stereotype.Component;  
  
@Component  
public class StudentFactoryBean implements FactoryBean<Student> {  
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
        return "StudentFactoryBean{}";  
    }  
}
```


首先来看当前是 FactoryBean 的情况：
```java
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
```

这里先调用 `getBean(FACTORY_BEAN_PREFIX + beanName)` 方法创建 FactoryBean 对象，注意这里先创建的是 FactoryBean 对象，

新增断点如图所示：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102343254.png)

可以看到当前的 `beanName` 为 `studentFactoryBean`，创建出来的 Object 对象是类型是 `StudentFactoryBean`。

所以当 Bean 是一个 FactoryBean 的时候，流程如下：
1. 传入一个特殊的 FactoryBean 的 BeanName；
2. 调用 `getBean`  方法得到一个实例化后的 FactoryBean 对象；
3. 判断 FactoryBean 对象是否实现了接口 SmartFactoryBean
    - 如果是则调用 `getObject()`  方法立马进行 FactoryBean 生产的对象创建；
    - 如果不是则跳过。

### getBean-实例化Bean对象

普通Bean的流程比较简单，直接调用 `getBean`  方法来获取一个实例化的 Bean 对象，

跟进代码，可以看到，这里最终调用的是  `AbstractBeanFactory#getBean` ：
```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```

`doGetBean`  源码较长，如图所示：
![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202210141643939.png)

这个方法就是 Bean 最终实例化的逻辑，流程很长，Bean 生命周期的绝大部分操作都是在这个方法里面完成的，后续再详细分析。
**这里我们就先直接理解为 `getBean`  方法就是通过 BeanName 得到一个实例化后的 Bean。**

## SmartInitializingSingleton接口

完成 Bean 的创建了，那就到了最后一步了，回调`SmartInitializingSingleton#afterSingletonsInstantiated()`方法。
```java
/**  
 * Trigger post-initialization callback for all applicable beans... 
 * 在创建了所有的Bean之后，遍历为所有适用的 bean 触发初始化后回调，  
 * 也就是这里会对延迟初始化的bean进行加载...  
 */
 for (String beanName : beanNames) {  
    /** 这一步其实是从缓存中获取对应的创建的Bean，这里获取到的必定是单例的 */  
    Object singletonInstance = getSingleton(beanName);  
    /**  
     * 判断单例bean是不是实现了SmartInitializingSingleton接口  
     * 所有的单例bean都创建后再调用SmartInitializingSingleton  
     */    
     if (singletonInstance instanceof SmartInitializingSingleton) {  
        /*** 记录执行时间，可以忽略 */  
        StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")  
            .tag("beanName", beanName);  
        SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;  
        /** spring安全管理器的判断，可以忽略 */  
        if (System.getSecurityManager() != null) {  
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {  
                smartSingleton.afterSingletonsInstantiated();  
                return null;            
            }, getAccessControlContext());  
        } else {  
            /**  
             * 调用了单例bean对象的afterSingletonsInstantiated方法  
             * 所以afterSingletonsInstantiated这个方法的调用时间是所有的非懒加载单例bean创建之后  
             */  
            smartSingleton.afterSingletonsInstantiated();  
        }  
        smartInitialize.end();  
    }  
}
```

SmartInitializingSingleton 是 Spring 4.1 中引入的新特性，与 InitializingBean 的功能类似，都是 Bean 实例化后执行自定义初始化，都是属于 Spring Bean 生命周期的增强。但是，SmartInitializingSingleton 的定义及触发方式方式上有些区别，它的定义不在当前的 Bean 中（a bean's local construction phase）。

InitializingBean 是在每一个 Bean 初始化完成后调用；多例的情况下每初始化一次就掉用一次。
**SmartInitializingSingleton 是所有的非延迟的、单例的 Bean 都初始化后调用，只调用一次。如果是多例的 Bean实现，不会调用。**

说明：
1. 实现 SmartInitializingSingleton的 Bean 需要是单例；
2. 在所有非延迟的单例的 Bean 初始化完成后调用。

SmartInitializingSingleton 有很多实现类，比较经典的是 `EventListenerMethodProcessor` 类，该类会在这一步完成了对已经创建好的 Bean 的解析，会判断其方法上是否有 `@EventListener` 注解，会将这个注解标注的方法通过 `EventListenerFactory` 转换成一个事件监听器并添加到监听器的集合中。