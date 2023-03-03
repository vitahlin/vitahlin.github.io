---
title: Spring源码6-BeanDefinition合并
<!-- description: 这是一个副标题 -->
date: 2023-01-09
slug: 01GTK062GAEYC6F3AWDFPG923F
categories:
    - Spring
tags:
    - Spring
---

## 前言

前文分析了 Spring 的 Bean 扫描流程的源码，扫描完成后已经得到了一个 BeanDefinition 集合，接下来就是遍历 BeanDefinition 列表，进行 Bean 的实例化。

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

## preInstantiateSingletons-Bean的实例化真正入口

跟进方法 `preInstantiateSingletons` 来看实例化的具体逻辑，`DefaultListableBeanFactory#preInstantiateSingletons` ：
![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2023/202301102210871.png)

这里主要有3部分逻辑，分别对应其中三个箭头：
1. BeanDefnition的合并-getMergedLocalBeanDefinition
2. BeanDefnition的实例化-getBean
3. SmartInitializingSingleton接口的实现

接下来，我们就来逐步看这三个步骤，本文重点介绍 BeanDefinition 的合并流程。

## getMergedLocalBeanDefinition-BeanDefinition合并

根据之前扫描得到 Bean 名称集合生成一个新的 Bean 名称数组，然后逐个遍历开始 Bean 的实例化。

首先，就是根据 `BeanDefinition`  生成 `RootBeanDefinition`  对象：
```java
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
```

在 Spring 中，一个 BeanDefinition 是可以设置父 BeanDefinition 的，仅需要调用其 `setParentName`   即可，之所以出现父子 Bean 是因为 Spring 允许将相同 Bean 的定义给抽出来，成为一个父 BeanDefinition，这样其它的 BeanDefinition 就能共用这些公共的数据了，**Bean有层次关系，子类需要合并父类的属性方法**。Spring 中支持父子 BeanDefinition，和 Java 父子类类似，但是完全不是一回事。

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


### 没有父BeanDefinition的情况

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
```

如果一个 `BeanDefinition` 有父类，那么Spring就会获取到父类的 `BeanName` (因为可能存在别名, 所以调用了 `transformedBeanName方法` )。

如果父类的 `BeanName` 和当前  `BeanName` 不一样，说明是存在真正的父类的。这个判断是用来防止我们将当前 `BeanName` 设置为当前 `BeanDefinition` 的父类而导致死循环的。

如果真正存在父类，Spring就会先将父类合并了，因为父类可能还有父类，该递归方法结束后就能获取到一个完整的父 BeanDefinition 了，然后 `new`  了一个 `RootBeanDefinition` ，将完整的父 BeanDefinition 放入进去，从而初步完成了合并。

执行后，我们就得到了**最终的合并之后的 BeanDefinition，放到了 Map 对象 `mergedBeanDefinitions` 中，后续都是从 `mergedBeanDefinitions` 拿合并后的 BeanDefiniton 进行实例化。**