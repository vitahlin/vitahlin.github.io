---
title: Spring源码6-Bean生命周期之Bean实例化(上)
description: Bean实例化
date: 2023-01-11
slug: spring-bean-instantiation-first
categories:
    - spring

tags:
    - spring
    - java
---


## 前言

前文分析了 Spring 的 Bean 扫描流程的源码，扫描完成后已经得到了一个 BeanDefinition 集合，接下来就是遍历扫描得到的 BeanDefinition 列表，进行 Bean 的实例化。

## Spring上下文初始化

首先，来看 Spring 中容器的初始化：
```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {  
    this();  
    register(componentClasses);  
    refresh();  
}
```

其中，Bean 的实例化流程是在 `refresh()`  方法中处理的，跟进方法`AnnotationConfigApplicationContext#refresh`：
```java
@Override  
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
        prepareRefresh();  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
        prepareBeanFactory(beanFactory);  
        try {  
            postProcessBeanFactory(beanFactory);  
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
            
            invokeBeanFactoryPostProcessors(beanFactory);  
            
            registerBeanPostProcessors(beanFactory);  
            beanPostProcess.end();  
            initMessageSource();  
            initApplicationEventMulticaster();  
             onRefresh();  
            registerListeners();  
            
            finishBeanFactoryInitialization(beanFactory);  
            
            finishRefresh();  
        } catch (BeansException ex) {  
            if (logger.isWarnEnabled()) {  
                logger.warn("Exception encountered during context initialization - " +  
                    "cancelling refresh attempt: " + ex);  
            }  
            destroyBeans();  
            cancelRefresh(ex);  
            throw ex;  
        } finally {  
            contextRefresh.end();  
        }  
    }  
}
```

其中，Bean 的实例化是在 `finishBeanFactoryInitialization(beanFactory)` 中完成的，`AbstractApplicationContext#finishBeanFactoryInitialization` 源码如下：
```java
 protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
            beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
        }
        
        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
        }
        
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }
        
        beanFactory.setTempClassLoader(null);
        
        beanFactory.freezeConfiguration();
        
        beanFactory.preInstantiateSingletons();
    }
```

生成非懒加载的单例 Bean 的处理逻辑在最后一行 `beanFactory.preInstantiateSingletons()`，实际上调用的是 `DefaultListableBeanFactory#preInstantiateSingletons`，看这方法命名就知道，实际上这个方法处理了单例 Bean 的实例化。

## Bean的实例化真正入口-preInstantiateSingletons

跟进方法 `preInstantiateSingletons` 来看实例化的具体逻辑，`DefaultListableBeanFactory#preInstantiateSingletons` ：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102210871.png)

这里主要有3部分逻辑，分别对应其中三个箭头：
1. BeanDefnition的合并-getMergedLocalBeanDefinition
2. BeanDefnition的实例化-getBean
3. SmartInitializingSingleton接口的实现

接下来，我们就来逐步看这三个步骤。

## 合并BeanDefinition

根据之前扫描得到 Bean 名称集合生成一个新的 Bean 名称数组，然后逐个遍历开始Bean 的实例化。

首先，就是根据 `BeanDefinition`  生成 `RootBeanDefinition`  对象：
```java
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
```

在 Spring 中，一个 BeanDefinition 是可以设置父 BeanDefinition 的，仅仅需要调用其 `setParentName`   即可，之所以出现父子 Bean 是因为 Spring 允许将相同 Bean 的定义给抽出来，成为一个父 BeanDefinition，这样其它的 BeanDefinition 就能共用这些公共的数据了，**Bean有层次关系，子类需要合并父类的属性方法**。Spring 中支持父子 BeanDefinition，和 Java 父子类类似，但是完全不是一回事。

例如：
```xml
<bean id="parent" class="com.vitah.service.Parent" scope="prototype"/>
<bean id="child" class="com.vitah.service.Child"/>
```
这么定义的情况下，child 是单例 Bean。

```xml
<bean id="parent" class="com.vitah.service.Parent" scope="prototype"/>
<bean id="child" class="com.vitah.service.Child" parent="parent"/>
```
但是这么定义的情况下，child 就是原型 Bean 了。

因为 child 的父 BeanDefinition 是 parent，所以会继承 parent 上所定义的 scope 属性。

Bean 定义公共的抽象类是 `AbstractBeanDefinition` ，普通的 Bean 在 Spring 加载 Bean 定义的时候，实例化出来的是`GenericBeanDefinition` ，而 Spring 上下文包括实例化所有 Bean 用的 AbstractBeanDefinition 是 RootBeanDefinition ，这时候就使用 `getMergedLocalBeanDefinition` 方法做了一次转化，将非 `RootBeanDefinition` 转换为` RootBeanDefinition` 以供后续操作。

跟进方法 `AbstractBeanFactory#getMergedLocalBeanDefinition` :
```java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
		RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
		if (mbd != null && !mbd.stale) {
			return mbd;
		}
		return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
	}
```
`mergedBeanDefinitions` 集合会先判断合并之后的 BeanDefinition 是否已经存在了，存在了直接返回。

不存在则继续跟进方法 `getMergedBeanDefinition` ，最终调用的方法是 `AbstractBeanFactory#getMergedBeanDefinition` ：
```java
protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
			throws BeanDefinitionStoreException {

		synchronized (this.mergedBeanDefinitions) {
			/*** 准备一个RootBeanDefinition变量引用，用于记录要构建和最终要返回的BeanDefinition */
			RootBeanDefinition mbd = null;
			RootBeanDefinition previous = null;

			// Check with full lock now in order to enforce the same merged instance.
			if (containingBd == null) {
				mbd = this.mergedBeanDefinitions.get(beanName);
			}

			if (mbd == null || mbd.stale) {
				previous = mbd;
				/*** 判读有没有指定parent */
				if (bd.getParentName() == null) {
					/**
					 * Use copy of given root bean definition.
					 * 如果自己就是一个RootBeanDefinition，那么就克隆一下，生成一个新的RootBeanDefinition
					 */
					if (bd instanceof RootBeanDefinition) {
						mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
					} else {
						mbd = new RootBeanDefinition(bd);
					}
				}
				else {
					/**
					 * Child bean definition: needs to be merged with parent.
					 * 定义父亲的BeanDefinition
					 */
					BeanDefinition pbd;
					try {
						/*** 拿到这个bean的parent的名字 */
						String parentBeanName = transformedBeanName(bd.getParentName());
						
						/*** 当前名字不等于parentBean的名字，递归处理，一直往上找 */
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
					/*** 生成一个新的BeanDefinition，并且覆盖掉父亲的属性 */
					mbd = new RootBeanDefinition(pbd);
					mbd.overrideFrom(bd);
				}

				// Set default singleton scope, if not configured before.
				if (!StringUtils.hasLength(mbd.getScope())) {
					mbd.setScope(SCOPE_SINGLETON);
				}

				// A bean contained in a non-singleton bean cannot be a singleton itself.
				// Let's correct this on the fly here, since this might be the result of
				// parent-child merging for the outer bean, in which case the original inner bean
				// definition will not have inherited the merged outer bean's singleton status.
				if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
					mbd.setScope(containingBd.getScope());
				}

				// Cache the merged bean definition for the time being
				// (it might still get re-merged later on in order to pick up metadata changes)
				if (containingBd == null && isCacheBeanMetadata()) {
					/*** 放到map中 */
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


### 没有父类BeanDefinition的情况

源码如下：
```java
if (bd.getParentName() == null) {  
    /**  
     * Use copy of given root bean definition.     
     * 如果自己就是一个RootBeanDefinition，那么就克隆一下，生成一个新的RootBeanDefinition  
     */    
    if (bd instanceof RootBeanDefinition) {  
        mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();  
    } else {  
        mbd = new RootBeanDefinition(bd);  
    }  
}
```

如果一个 BeanDefinition 没有设置 `parentName` ，表示该 BeanDefinition 是没有父BeanDefinition 的。
此时，如果这个 BeanDefinition 是 RootBeanDefinition 的实例或子类实例，Spring就会直接克隆一个一模一样的 BeanDefinition，`cloneBeanDefinition` 方法的代码很简单，即 `new RootBeanDefinition(this)` 。 
如果不是 RootBeanDefinition 类型，则创建一个 RootBeanDefinition 并将当前bd合并进去。

这时候肯定会有疑问，两个合并的代码不是一模一样吗？
其实不是，如果一个 BeanDefinition 是 RootBeanDefinition 的子类，那么调用克隆方法的时候，就会 new 一个该子类，然后开始克隆，比如 `new ConfigurationClassBeanDefinition(this)`。

### 有父BeanDefinition的情况

```java
 else {  
    /**  
     * Child bean definition: needs to be merged with parent.     * 定义父亲的BeanDefinition  
     */    BeanDefinition pbd;  
    try {  
        /*** 拿到这个bean的parent的名字 */  
        String parentBeanName = transformedBeanName(bd.getParentName());  
        /*** 当前名字不等于parentBean的名字，递归处理，一直往上找 */  
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
    /*** 生成一个新的BeanDefinition，并且覆盖掉父亲的属性 */  
    mbd = new RootBeanDefinition(pbd);  
    mbd.overrideFrom(bd);  
}
```

如果一个 `BeanDefinition` 有父类，那么Spring就会获取到父类的 `BeanName` (因为可能存在别名, 所以调用了 `transformedBeanName方法` )。

如果父类的 `BeanName` 和当前  `BeanName` 不一样，说明是存在真正的父类的。这个判断是用来防止我们将当前 `BeanName` 设置为当前 `BeanDefinition` 的父类而导致死循环的。

如果真正存在父类，Spring就会先将父类合并了，因为父类可能还有父类，该递归方法结束后就能获取到一个完整的父 BeanDefinition 了，然后 `new`  了一个 `RootBeanDefinition` ，将完整的父 BeanDefinition 放入进去，从而初步完成了合并。

执行后，我们就得到了**最终的合并之后的 BeanDefinition，放到了 Map 对象 `mergedBeanDefinitions` 中，后续都是从 `mergedBeanDefinitions` 拿合并后的 BeanDefiniton 进行实例化。**

## Bean的实例化

获取 `RootBeanDefinition` 完成后，接下来就调用 `getBean(beanName)` 进行 Bean 的实例化了。当然这里的创建还是分了**工厂Bean**和**非工厂Bean**两个逻辑

- 如果不是 FactoryBean 的话，那么很简单直接调用 `getBean` 方法 获取实例化后的 Bean 对象
- 如果是一个工厂 Bean，那么`getBean(beanName)` 这一步只会创建一个工厂 Bean，接下来会通过`isEagerInit` 参数判断是否需要初始化工厂 Bean 的对象，如果需要，再调用`getBean(beanName)` 去立马获取工厂Bean需要生产的对象。我们可以新建测试类来验证。

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
3. 判断 FactoryBean 对象是否实现了接口 SmartFactoryBean，
    - 如果是则调用 `getObject()`  方法立马进行 FactoryBean 生产的对象创建；
    - 如果不是则跳过。

### getBean 

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

这个方法就是 Bean 最终实例化的逻辑，流程很长，Bean 生命周期的绝大部分操作都是在这个方法里面完成的，后续再详细分析。这里我们就先直接理解为 `getBean`  方法就是通过 BeanName 得到一个实例化后的 Bean。

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
**SmartInitializingSingleton 是所有的非延迟的、单例的 Bean 都初始化后调用，只调用一次。**如果是多例的 Bean实现，不会调用。

说明：
1. 实现 SmartInitializingSingleton的 Bean 需要是单例；
2. 在所有非延迟的单例的 Bean 初始化完成后调用。

SmartInitializingSingleton 有很多实现类，比较经典的是 `EventListenerMethodProcessor` 类，该类会在这一步完成了对已经创建好的 Bean 的解析，会判断其方法上是否有 `@EventListener` 注解，会将这个注解标注的方法通过 `EventListenerFactory` 转换成一个事件监听器并添加到监听器的集合中。