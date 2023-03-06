---
title: Spring源码9-Bean实例化之createBean
<!-- description: 这是一个副标题 -->
date: 2023-03-06
lastmod: 2023-03-06
slug: 01GTV8VYZHP32YMC891E9MJWTD
categories:
    - Spring
tags:
    - Spring
---

## 前言

前文已经介绍了 Spring 中 Bean 的实例化其实执行的是一个 lambda 表达式，即通过执行 createBean 方法创建 Bean 对象，接下来就来分析 createBean 方法的源码。

## createBean方法

直接跟进 `createBean`  方法的代码，`AbstractAutowireCapableBeanFactory#createBean` :
```java
@Override  
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)  
    throws BeanCreationException {  
    /*** 日志打印 */  
    if (logger.isTraceEnabled()) {  
        logger.trace("Creating instance of bean '" + beanName + "'");  
    }  
    /*** 使用的BeanDefinition */  
    RootBeanDefinition mbdToUse = mbd;  
    /**  
     * 这里进行类的加载，马上就要实例化bean了，确保beanClass被加载了.
     * 根据设置的 class 属性或者根据 className 来解析 Class,  
     * 解析得到beanClass，为什么需要解析呢？如果是从XML中解析出来的标签属性肯定是个字符串,
     * 所以这里需要加载类，得到Class对象.
     */  
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);  
    /**  
     * bean被加载，并且当前BeanDefinition的beanClass属性不是Class类型，  
     * 并且对应的beanName不是null  
     */    
     if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {  
        mbdToUse = new RootBeanDefinition(mbd);  
        /**  
         * 拿到类以后，就设置BeanDefinition的beanClass属性，把类设置进去  
         * 在这里，beanClass的类型就从String变为Class  
         */        
         mbdToUse.setBeanClass(resolvedClass);  
    }  
    /**  
     * Prepare method overrides.     
     * 这里主要处理LookUp注解，进行方法的替代。  
     * 对于@LookUp注解标注的方法是不需要在这里处理的，AutowiredAnnotationBeanPostProcessor会处理这个注解  
     */  
    try {  
        mbdToUse.prepareMethodOverrides();  
    } catch (BeanDefinitionValidationException ex) {  
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),  
            beanName, "Validation of method overrides failed", ex);  
    }  
    /*** 到这里，已经完成了类的加载，接着就需要进行类的实例化 */  
    try {  
        /**  
         * Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.         
         * 实例化前的处理，如果当前bean实现了InstantiationAwareBeanPostProcessor，判断是否有返回值  
         * 返回值不为null，则用返回的对象替代真正的bean实例  
         *  
         * 在实例化对象前，会经过后置处理器处理，  
         * 这个后置处理器的提供了一个短路机制，可以提前结束整个Bean的生命周期，直接从这里返回一个Bean。  
         * 不过我们一般不会这么做，它的另外一个作用就是对AOP提供了支持。  
         * 在这里会将一些不需要被代理的Bean进行标记  
         */  
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);  
        /**  
         * 若果有自定义bean则直接返回了bean，不会再走后续的doCreateBean方法。  
         * 这里并不限制返回值的类型  
         */  
        if (bean != null) {  
            return bean;  
        }  
    } catch (Throwable ex) {  
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,  
            "BeanPostProcessor before instantiation of bean failed", ex);  
    }  
    try {  
        /**  
         * bean不存在提前初始化的操作，则开始正常的创建流程  
         * doXXX方法，真正干活的方法，doCreateBean，真正创建Bean的方法，创建Bean实例  
         */  
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);  
        if (logger.isTraceEnabled()) {  
            logger.trace("Finished creating instance of bean '" + beanName + "'");  
        }  
        return beanInstance;  
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {  
        // A previously detected exception with proper bean creation context already,  
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.        throw ex;  
    } catch (Throwable ex) {  
        throw new BeanCreationException(  
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);  
    }  
}
```

createBean 的整个流程可分为4个小方法：
1. resolveBeanClass：加载 Class 对象
2. prepareMethodOverrides：处理 `LookUp`  注解
3. resolveBeforeInstantiation：实例化前的应用后置处理器
4. doCreateBean：创建 Bean

### resolveBeanClass-类加载

`AbstractBeanFactory#resolveBeanClass`：
```java
/**  
 * 判断有没有beanClass，这里实际上判断的是beanClass是不是Class对象  
 * 返回false就表示还没有加载，true就表示已经加载，已经加载则直接返回。  
 * 在bean扫描里面，beanClass最开始是一个字符串类型，值是className  
 */
 if (mbd.hasBeanClass()) {  
    return mbd.getBeanClass();  
}  
  
/*** 如果beanClass没有被加载，进行bean的加载 */  
if (System.getSecurityManager() != null) {  
    /**  
     * AccessController.doPrivileged:允许在一个类实例中的代码通知这个AccessController：  
     * 它的代码主体是享受"privileged(特权的)"，它单独负责对它的可得的资源的访问请求，  
     * 而不管这个请求是由什么代码所引发的。  
     * 一个调用者在调用doPrivileged方法时，可被标识为 "特权"。在做访问控制决策时，  
     * 如果checkPermission方法遇到一个通过doPrivileged调用而被表示为 "特权"的调用者，  
     * 并且没有上下文自变量，checkPermission方法则将终止检查。如果那个调用者的域具有特定的许可，  
     * 则不做进一步检查，checkPermission安静地返回，表示那个访问请求是被允许的；  
     * 如果那个域没有特定的许可，则象通常一样，一个异常被抛出。  
     * 参考博客：https://www.jianshu.com/p/3fe79e24f8a1  
     * 参考博客：https://www.iteye.com/blog/huangyunbin-1942509  
     */    
     return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>)  
        () -> doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());  
} else {  
    /*** 这里就是加载当前BeanDefinition所对应的类 */  
    return doResolveBeanClass(mbd, typesToMatch);  
}
```

`resolveBeanClass`  方法的是为 `RootBeanDefinition`  解析出对应的 Bean Class，流程如下：
1. 如果 mbd 指定了 bean class，就直接返回该 bean class
2. 调用 `doResolveBeanClass(mbd, typesToMatch)` 来解析获取对应的 bean Class 对象然后返回出去。如果成功获取到系统的安全管理器，使用特权的方式调用。
3. 捕捉 PrivilegedActionException, ClassNotFoundException 异常和 LinkageError 错误，保证其异常或错误信息， 抛出 CannotLoadBeanClassException。

核心逻辑是方法 `doResolveBeanClass(mbd, typesToMatch)` ，继续看 `doResolveBeanClass`  方法的源代码
`AbstractBeanFactory#doResolveBeanClass` ：
```java
@Nullable  
private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)  
    throws ClassNotFoundException {  
    /*** 得到当前bean的类加载器 */  
    ClassLoader beanClassLoader = getBeanClassLoader();  
    /*** 定义一个动态的类加载器。有临时类加载器用临时类加载器，没有临时类加载器用现在这个 */  
    ClassLoader dynamicLoader = beanClassLoader;  
    /*** 表示RootBeanDefinition的配置的bean类名需要重新被dynameicLoader加载的标记，默认不需要 */  
    boolean freshResolve = false;  
    
    /*** 进行类型匹配 */  
    if (!ObjectUtils.isEmpty(typesToMatch)) {  
        /**  
         * When just doing type checks (i.e. not creating an actual instance yet),         
         * use the specified temporary class loader (e.g. in a weaving scenario).         
         * 当只是进行类型检查（即尚未创建实际实例）时，请使用指定的临时类加载器（例如在编织场景中）  
         */  
        ClassLoader tempClassLoader = getTempClassLoader();  
        if (tempClassLoader != null) {  
            dynamicLoader = tempClassLoader;  
            /*** 标记当前的RootBeanDefinition需要被dynamicLoader解析 */  
            freshResolve = true;  
            /*** 他是spring自定义的DecoratingClassLoader类加载器 */  
            if (tempClassLoader instanceof DecoratingClassLoader) {  
                /*** 类加载器强制转换 */  
                DecoratingClassLoader dcl = (DecoratingClassLoader) tempClassLoader;  
                /*** 类型匹配 */  
                for (Class<?> typeToMatch : typesToMatch) {  
                    dcl.excludeClass(typeToMatch.getName());  
                }  
            }  
        }  
    }  
    /*** 获取className：beanClass是名称直接返回，是class类型返回class对应的Name */  
    String className = mbd.getBeanClassName();  
    if (className != null) {  
        /**  
         * 这里判断当前beanName是不是一个spring表达式  
         * 用xml定义bean的时候，class属性可以设置spring表达式，这里就是处理这个逻辑。  
         * 如果是spring表达式，调用spring表达式解析器进行解析  
         */  
        Object evaluated = evaluateBeanDefinitionString(className, mbd);  
        if (!className.equals(evaluated)) {  
            // A dynamically resolved expression, supported as of 4.2...  
            if (evaluated instanceof Class) {  
                return (Class<?>) evaluated;  
            } else if (evaluated instanceof String) {  
                className = (String) evaluated;  
                freshResolve = true;  
            } else {  
                throw new IllegalStateException("Invalid class name expression result: " + evaluated);  
            }  
        }  
        /*** 类没有被加载的时候，进行加载 */  
        if (freshResolve) {  
            /**  
             * When resolving against a temporary class loader, exit early in order             
             * to avoid storing the resolved Class in the bean definition.             
             * 类加载器不能为空  
             */  
            if (dynamicLoader != null) {  
                try {  
                    /*** 返回用dynamicLoader具体的类加载器加载类，得到class对象 */  
                    return dynamicLoader.loadClass(className);  
                } catch (ClassNotFoundException ex) {  
                    if (logger.isTraceEnabled()) {  
                        logger.trace("Could not load class [" + className + "] from " + dynamicLoader + ": " + ex);  
                    }  
                }  
            }  
            /*** 加载数组，String，基本类型 */  
            return ClassUtils.forName(className, dynamicLoader);  
        }  
    }  
    /**  
     * Resolve regularly, caching the result in the BeanDefinition... 
     * 正常解析类，给之前存储BeanName的属性重新赋值为Class对象。
     */  
    return mbd.resolveBeanClass(beanClassLoader);  
}
```

其中第一行 `ClassLoader beanClassLoader = getBeanClassLoader()` ，是用来获取当前的类加载器，最终的执行方法是 `ClassUtils.getDefaultClassLoader`，看这个方法命名就知道方法是用来获取当前的默认类加载器。继续看源码：
```java
@Nullable  
public static ClassLoader getDefaultClassLoader() {  
    ClassLoader cl = null;  
    try {  
        /**  
         * 获取当前线程的默认的类加载器  
         * 应用场景：tomcat中会设置每个线程的类加载器  
         */  
        cl = Thread.currentThread().getContextClassLoader();  
    } catch (Throwable ex) {  
        // Cannot access thread context ClassLoader - falling back...  
    }  
    if (cl == null) {  
        // No thread context class loader -> use class loader of this class.  
        /**  
         * 当前线程中没有类加载器  
         * ClassUtils这个类是被哪个类加载器加载的，就获取对应的这个加载器  
         */  
        cl = ClassUtils.class.getClassLoader();  
        if (cl == null) {  
            // getClassLoader() returning null indicates the bootstrap ClassLoader  
            try {  
                cl = ClassLoader.getSystemClassLoader();  
            } catch (Throwable ex) {  
                // Cannot access system ClassLoader - oh well, maybe the caller can live with null...  
            }  
        }  
    }  
    return cl;  
}
```
首先默认获取当前线程的类加载器，如果没有就获取系统的类加载器。

`doResolveBeanClass`  方法的逻辑如下：
1. 先获取类加载器；
2. 然后这里传入的`typesToMatch`参数对象数组为空，所以不会走排除部分类的逻辑；
3. 接下来是使用`evaluateBeanDefinitionString()` 方法计算表达式，如果传入的 `className` 有占位符，会在这里被解析；
4. 最终正常我们会走到`mbd.resolveBeanClass(beanClassLoader)`方法里，把Class对象返回


然后回到 createBean 方法：
```java
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {  
    mbdToUse = new RootBeanDefinition(mbd);  
    /**  
     * 拿到类以后，就设置BeanDefinition的beanClass属性，把类设置进去  
     * 在这里，beanClass的类型就从String变为Class  
     */    
     mbdToUse.setBeanClass(resolvedClass);  
}
```

**Class 加载后，就把这个类赋值给 BeanDefinition 的 beanClass 属性，从这里开始，beanClass 属性类型就从原先的 String 变为 Class。**


### prepareMethodOverrides-处理LookUp注解

这个方法主要是处理 `LookUp`  注解。

《Spirng源码深度解析》里面有一段话：
> 很多读者可能会不知道这个方法的作用，因为在 Spring 的配置里面根本就没有诸如 `override-method` 之类的配置， 那么这个方法到底是干什么用的呢？ 其实在 Spring 中确实没有 `override-method` 这样的配置，但是在 Spring 配置中是存在 `lookup-method` 和 `replace-method` 的，而这两个配置的加载其实就是将配置统一存放在 `BeanDefinition` 中的 `methodOverrides` 属性里，而这个函数的操作其实也就是针对于这两个配置的。

`lookup-method` 通常称为获取器注入，这是一种特殊的方法注入，它是把一个方法声明为返回某种类型的 Bean，而实际要返回的 Bean 是在配置文件里面配置的，可用在设计可插拔的功能上，解除程序依赖。 这里会对一些重载方法进行标记预处理，如果同方法名的方法只存在一个，那么会将覆盖标记为未重载，以避免 arg 类型检查的开销。

这个操作我们平常比较少用，这里不做过多的介绍。

### resolveBeforeInstantiation-实例化前

在Spring中，实例化对象之前，Spring提供了一个扩展点，允许用户来控制是否在某个或某些Bean实例化之前做一些启动动作。这个扩展点叫 **InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()**。比如：
```java
@Component
public class MyBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
		}
		return null;
	}
}
```
如上代码会导致，在 `userService` 这个 Bean 实例化前，会进行打印。

值得注意的是，`postProcessBeforeInstantiation()` 是有返回值的，如果这么实现：
```java
@Component
public class MyBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
			return new UserService();
		}
		return null;
	}
}
```
`userService` 这个Bean，在实例化前会直接返回一个由我们所定义的 `UserService` 对象。如果是这样，表示不需要Spring来实例化了，并且后续的Spring依赖注入也不会进行了，会跳过一些步骤，直接执行初始化后这一步。

不过我们一般不会这么做，它的另外一个作用就是对 AOP 提供了支持，我们可以暂时理解它没有起到任何作用。

跟进方法 `AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation`  查看：
```java
@Nullable  
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {  
    Object bean = null;  
    /*** 如果当前还没有进行实例化前的操作 */  
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {  
        /**  
         * Make sure bean class is actually resolved at this point.         
         * synthetic表示合成  
         * 这里判断mbd不是合成的，并且BeanPostProcessors缓存中存在InstantiationAwareBeanPostProcessor  
         */        
         if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
            /**  
             * determineTargetType推断当前的类，得到Class类型，即解析beanName对应的Bean实例的类型  
             */  
            Class<?> targetType = determineTargetType(beanName, mbd);  
            if (targetType != null) {  
                /**  
                 * 这里就是执行InstantiationAwareBeanPostProcessor  
                 * 提供一个提前初始化的时机，这里会直接返回一个实例对象  
                 */  
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);  
                if (bean != null) {  
                    /**  
                     * 如果提前初始化成功，则执行postProcessAfterInitialization() 方法，
                     * 注意单词Instantiation和Initialization区别。  
                     * 
                     * 关于这一块的逻辑，这里漏了实例化后置处理、初始化前置处理这两个方法。  
                     * 而是在提前返回对象后，直接执行了初始化后置处理器就完成了Bean的整个流程，  
                     * 相当于是提供了一个短路的操作，不再经过Spring提供的繁杂的各种处理。  
                     *  
                     * 对象已经在这里进行提前初始化，初始化完成后我们又不执行Spring的复杂处理流程，  
                     * 只能直接在这里直接执行后置处理器，不然后置处理器就相当于被跳过，没有执行了。  
                     */  
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);  
                }  
            }  
        }  
        /*** 如果bean不为空，beforeInstantiationResolved设为true，代表在实例化前已经解析 */  
        mbd.beforeInstantiationResolved = (bean != null);  
    }  
    return bean;  
}
```

跟进代码 `applyBeanPostProcessorsBeforeInstantiation(targetType, beanName)` 查看，这里只要有一个 `InstantiationAwareBeanPostProcessor` 返回的结果不为空，则直接返回，说明多个 `InstantiationAwareBeanPostProcessor` 只会生效靠前的一个，这里其实Spring会对多个 BeanPostProocessor 进行排序，后续在 Spring 启动流程中详细说明。

```java
@Nullable  
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {  
    /*** 获取所有的InstantiationAwareBeanPostProcessor，然后一个个调用 */  
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
        /**  
         * 这里调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法  
         * 如果有返回值，直接返回，如果没有返回值则继续调用。而且返回值并不要求是具体某一个类型  
         * 说明多个 InstantiationAwareBeanPostProcessor 只会生效靠前的一个。
         * 这里其实BeanPostProcessor会进行排序，后续在Spring启动流程中说明。
         */  
        Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);  
        if (result != null) {  
            return result;  
        }  
    }  
    return null;  
}
```

**注意：并不要求一定要返回当前这个类型的Bean对象，直接返回一个其他的对象也可以，因为这里的返回值是 Object 类型。** 

当有返回值时，继续执行 `applyBeanPostProcessorsAfterInitialization(bean, beanName)` ，跟进方法，逻辑跟上面的是类似的，**注意单词Instantiation和Initialization区别**。
```java
@Override  
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)  
    throws BeansException {  
    Object result = existingBean;  
    for (BeanPostProcessor processor : getBeanPostProcessors()) {  
        Object current = processor.postProcessAfterInitialization(result, beanName);  
        if (current == null) {  
            /*** 有一个为空则也直接返回 */  
            return result;  
        }  
        result = current;  
    }  
    return result;  
}
```

## doCreateBean

经过上面的步骤，我们来到了`doCreateBean(beanName, mbdToUse, args)`方法，这是真正进行 Bean 创建的地方，所以这里才是真的进入正文，前面都是打酱油走走过程。

跟进 doCreateBean 方法的代码：
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)  
    throws BeanCreationException {  
    /**  
     * 这个方法真正创建了Bean,创建一个Bean会经过 创建对象->依赖注入->初始化，  
     * 这三个过程，在这个过程中，BeanPostProcessor会穿插执行，  
     */
       
    /**  
     * Instantiate the bean.     
     * 新建bean的包装类，这个到时候推断构造方法的时候来详细说明。  
     */  
    BeanWrapper instanceWrapper = null;  
    if (mbd.isSingleton()) {  
        /**  
         * 有可能在spring启动前，就需要把某些bean创建出来（比如依赖注入过程中）。  
         * 如果是factoryBean，则需要先移除未完成的FactoryBean实例的缓存  
         * 注意factoryBeanInstanceCache是ConcurrentMap，remove方法会返回删除的键值(如果不存在返回null).  
         * 我们平常也很少会用到这种场景.  
         */        
         instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);  
    }  
    if (instanceWrapper == null) {  
        /**  
         * 根据指定bean使用对应的策略创建新的实例，如工厂方法，构造函数自动注入，简单初始化。  
         * 这个方法里面会推断构造方法，真正的实例化对象。  
         */  
        instanceWrapper = createBeanInstance(beanName, mbd, args);  
    }  
    /*** 这里就能得到一个具体的Bean对象 */  
    Object bean = instanceWrapper.getWrappedInstance();  
    Class<?> beanType = instanceWrapper.getWrappedClass();  
    if (beanType != NullBean.class) {  
        mbd.resolvedTargetType = beanType;  
    }  
    // Allow post-processors to modify the merged bean definition.  
    /**  
     * 按照官方的注释来说，这个地方是Spring提供的一个扩展点，  
     * 对程序员而言，我们可以通过一个实现了MergedBeanDefinitionPostProcessor的后置处理器,  
     * 来修改bd中的属性,从而影响到后续的Bean的生命周期.  
     * 不过官方自己实现的后置处理器并没有去修改bd,而是调用了applyMergedBeanDefinitionPostProcessors方法.  
     * 这个方法名直译过来就是-应用合并后的bd,也就是说它这里只是对bd做了进一步的使用而没有真正的修改.  
     */    
     synchronized (mbd.postProcessingLock) {  
        /*** bd只允许被处理一次 */  
        if (!mbd.postProcessed) {  
            try {  
                /**  
                 * 处理MergeBeanDefinition的BeanDefinitionPostProcessors，  
                 * 这是实例化之后，属性赋值之前的一个步骤,其实是处理BeanDefinition的BeanPostProcessor.  
                 * 这里其实处理了@Autowired注解，后续依赖注入再分析。  
                 */  
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
            } catch (Throwable ex) {  
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                    "Post-processing of merged bean definition failed", ex);  
            }  
            mbd.postProcessed = true;  
        }  
    }  
    /**  
     * Eagerly cache singletons to be able to resolve circular references     
     * even when triggered by lifecycle interfaces like BeanFactoryAware.     
     * 这里是bean的循环依赖处理,后续分析  
     */  
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
        isSingletonCurrentlyInCreation(beanName));  
    if (earlySingletonExposure) {  
        if (logger.isTraceEnabled()) {  
            logger.trace("Eagerly caching bean '" + beanName +  
                "' to allow for resolving potential circular references");  
        }  
        /**  
         * 为避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂，  
         * getEarlyBeanReference对bean再一次依赖引用，主要应用SmartInstantiationAwareBeanPostProcessor，  
         * 其中我们熟悉的AOP就是在这里将advice动态织入bean中，若没有则直接返回bean，不做任何处理  
         */  
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));  
    }  
    // Initialize the bean instance.  
    Object exposedObject = bean;  
    try {  
        /*** 填充属性 */  
        populateBean(beanName, mbd, instanceWrapper);  
        /*** 初始化 */  
        exposedObject = initializeBean(beanName, exposedObject, mbd);  
    } catch (Throwable ex) {  
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {  
            throw (BeanCreationException) ex;  
        } else {  
            throw new BeanCreationException(  
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);  
        }  
    }  
    /*** 循环依赖的处理 */  
    if (earlySingletonExposure) {  
        Object earlySingletonReference = getSingleton(beanName, false);  
        if (earlySingletonReference != null) {  
            if (exposedObject == bean) {  
                exposedObject = earlySingletonReference;  
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {  
                String[] dependentBeans = getDependentBeans(beanName);  
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);  
                for (String dependentBean : dependentBeans) {  
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                        actualDependentBeans.add(dependentBean);  
                    }  
                }  
                if (!actualDependentBeans.isEmpty()) {  
                    throw new BeanCurrentlyInCreationException(beanName,  
                        "Bean with name '" + beanName + "' has been injected into other beans [" +  
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +  
                            "] in its raw version as part of a circular reference, but has eventually been " +  
                            "wrapped. This means that said other beans do not use the final version of the " +  
                            "bean. This is often the result of over-eager type matching - consider using " +  
                            "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");  
                }  
            }  
        }  
    }  
    // Register bean as disposable.  
    try {  
        /*** 定义Bean销毁的一些东西 */  
        registerDisposableBeanIfNecessary(beanName, bean, mbd);  
    } catch (BeanDefinitionValidationException ex) {  
        throw new BeanCreationException(  
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
    }  
    /*** 这里就返回创建的Bean */  
    return exposedObject;  
}
```

然后我们就来逐步分析 doCreateBean 的流程。

### 1. BeanWrapper

新建一个 Bean 的包装类，这个到时候推断构造方法的时候来详细说明。  

### 2.  this.factoryBeanInstanceCache.remove(beanName)

首先如果是单例，会到`factoryBeanInstanceCache`中获取是否存在缓存，如果有这里就会从缓存里获取一个`instanceWrapper`，不需要再去走复杂的创建流程了。

这里 `factoryBeanInstanceCache`  这个集合什么时候会有值？我们用一个例子来说明这个流程。
```java
package org.springframework.vitahlin.bean;  
  
public class ProductService {  
    public void hello() {  
        System.out.println("This is productService");  
    }  
}
```

```java
package org.springframework.vitahlin.bean;  
  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Service;  
  
@Service  
public class OrderService {  
    @Autowired  
    private ProductService productService;  
}
```

```java
package org.springframework.vitahlin.bean;  
  
import org.springframework.beans.factory.FactoryBean;  
import org.springframework.context.annotation.DependsOn;  
import org.springframework.stereotype.Service;  
  
/**  
 * DependsOn注解表示了ProductFactoryBean要在的创建要在OrderService创建之后，  
 * 主要目的是为了让ProductFactoryBean在创建前让容器发生一次属性注入  
 */  
@Service  
@DependsOn("orderService")  
public class ProductFactoryBean implements FactoryBean<ProductService> {  
    @Override  
    public ProductService getObject() throws Exception {  
        return new ProductService();  
    }  
    @Override  
    public Class<?> getObjectType() {  
        return ProductService.class;  
    }  
}
```

测试用例中，我们明确表示了 `ProductFactoryBean` 是依赖于 `orderService` 的，所以必定会先创建 `orderService` 再创建 `ProductFactoryBean` ，创建 `orderService` 的流程如下：
1. 开始创建 Bean
2. 实例化一个 OrderService
3. 为 OrderService 进行属性注入
4. 为 OrderService 进行初始化
5. 结束


其中第三步的属性注入阶段，我们需要细化，流程图如下：

![](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202211172110252.png)

1. 找到需要注入的注入点，也就是 `orderService` 的 `productService` 字段；
2. 根据字段的类型以及名称去容器中查询符合要求的 `Bean` ；
3. 当遍历到一个 `FactoryBean`  时，为了确定其 `getObject`  方法返回的对象类型需要创建这个 `FactoryBean` (只会到对象级别)，然后调用这个创建好的 `FactoryBean` 的 `getObjectType`  方法明确其类型并于注入点需要的类型比较，看是否是一个候选的 Bean，在创建这个 `FactoryBean` 时就将其放入了 `factoryBeanInstanceCache` 中；
4. 确定了唯一的候选 Bean 之后，Spring 就会对这个 Bean 进行创建，因为此时 `factoryBeanInstanceCache`  已经缓存了这个 Bean 对应的对象，所以直接通过 `this.factoryBeanInstanceCache.remove(beanName)`  这行代码就返回了，避免二次创建对象。


### 3. createBeanInstance-Bean实例化

在这个步骤就会根据 BeanDefinition 去实例化对象了，来看方法源码：
```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {  
    /**  
     * Make sure bean class is actually resolved at this point.     
     * 确保此时已经解析了bean类  
     */  
    Class<?> beanClass = resolveBeanClass(mbd, beanName);  
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {  
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());  
    }  
    /**  
     * 通过bd中提供的instanceSupplier来获取一个对象。  
     * 正常bd中都不会有这个instanceSupplier属性，这里也是Spring提供的一个扩展点，但实际上不常用  
     */  
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();  
    if (instanceSupplier != null) {  
        return obtainFromSupplier(instanceSupplier, beanName);  
    }  
    /*** 实例化通过@Bean注解定义的Bean */  
    if (mbd.getFactoryMethodName() != null) {  
        return instantiateUsingFactoryMethod(beanName, mbd, args);  
    }  
    /*** 是否推断过构造函数 */  
    boolean resolved = false;  
    /*** 构造函数是否需要进行注入 */  
    boolean autowireNecessary = false;  
    if (args == null) {  
        synchronized (mbd.constructorArgumentLock) {  
            /**  
             * 一个类里面有多个构造函数，每个构造函数都有不同的参数，  
             * 所以调用前需要根据参数锁定要调用的构造函数或工厂方法  
             */  
            if (mbd.resolvedConstructorOrFactoryMethod != null) {  
                resolved = true;  
                autowireNecessary = mbd.constructorArgumentsResolved;  
            }  
        }  
    }  
    /*** 如果已经解析过则使用解析好的构造函数方法，不需要再次锁定 */  
    if (resolved) {  
        if (autowireNecessary) {  
            /*** 构造函数自动注入，这里就会推断构造方法 */  
            return autowireConstructor(beanName, mbd, null, null);  
        } else {  
            /*** 使用默认构造函数进行构造 */  
            return instantiateBean(beanName, mbd);  
        }  
    }  
    /**  
     * 需要根据参数解析构造函数  
     */  
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);  
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||  
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {  
        /*** 构造函数自动注入 */  
        return autowireConstructor(beanName, mbd, ctors, args);  
    }  
    /**  
     * 默认构造的首选构造函数？  
     */  
    ctors = mbd.getPreferredConstructors();  
    if (ctors != null) {  
        return autowireConstructor(beanName, mbd, ctors, null);  
    }  
    /**  
     * 使用默认构造函数  
     */  
    return instantiateBean(beanName, mbd);  
}
```

这个方法干了哪几件事？
-   首先尝试调用`obtainFromSupplier()`实例化 Bean
-   尝试调用`instantiateUsingFactoryMethod()`实例化 Bean
-   根据给定参数推断构造函数实例化 Bean
-   以上均无，则使用默认构造函数实例化 Bean

### 4. BeanDefinition的后置处理器-MergedBeanDefinitionPostProcessor

Bean对象实例化出来之后，接下来就应该给对象的属性赋值了。
在真正给属性赋值之前，Spring又提供了一个扩展点 **MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()** ，可以对此时的 BeanDefinition 进行加工，比如：
```java
@Component
public class MyMergedBeanDefinitionPostProcessor implements MergedBeanDefinitionPostProcessor {

	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		if ("userService".equals(beanName)) {
			beanDefinition.getPropertyValues().add("orderService", new OrderService());
		}
	}
}
```

在Spring源码中，AutowiredAnnotationBeanPostProcessor 就是一个 MergedBeanDefinitionPostProcessor，它的postProcessMergedBeanDefinition() 中会去查找注入点，并缓存在 AutowiredAnnotationBeanPostProcessor 对象的一个Map中（injectionMetadataCache）。

来看 `applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName)` 的源码，比较简单，就是获取所有的`MergedBeanDefinitionPostProcessor`，然后依次执行它的`postProcessMergedBeanDefinition()`方法：
```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {  
    /*** 就是调用了MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法 */  
    for (MergedBeanDefinitionPostProcessor processor : getBeanPostProcessorCache().mergedDefinition) {  
        processor.postProcessMergedBeanDefinition(mbd, beanType, beanName);  
    }  
}
```


### 5. 依赖处理

这部分主要是为了处理循环依赖而做的准备，这里会根据`earlySingletonExposure`参数去判断是否允许循环依赖，如果允许，则会调用`addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))`方法将bean的早期引用放入到`singletonFactories`中。关于循环依赖的详细处理过程，我们后续再详细讲解。
```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
    isSingletonCurrentlyInCreation(beanName));  
if (earlySingletonExposure) {  
    if (logger.isTraceEnabled()) {  
        logger.trace("Eagerly caching bean '" + beanName +  
            "' to allow for resolving potential circular references");  
    }  
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));  
}
```

### 6. populateBean-属性赋值

到这里会已经完成了 bean 的实例化，早期引用的暴露，那接下来就到了属性填充的部分，开始对 Bean 进行各种赋值，让一个空壳半成品 Bean 完善成一个有血有肉的正常 Bean。
这里可能存在依赖其他 Bean的属性，会递归初始化依赖 Bean。

populateBean 的总体流程：
1. 激活 `InstantiationAwareBeanPostProcessor`  后置处理器的 `postProcessAfterinstantiation` 方法；
2. 解析依赖注入的方式，将属性装配到 PropertyValues 中：resolvedAutowireMode；
3. 激活 `InstantiationAwareBeanPostProcessor#postProcessProperties` ：对@AutoWired 标记的属性进行依赖注入；
4. 依赖检查 checkDependencies；
5. 将解析的值用 BeanWrapper 进行包装 applyPropertyValues。

#### postProcessAfterinstantiation-实例化后处理

```java
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
        if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {  
            return;  
        }  
    }  
}
```

populateBean中，首先会经过判空校验，校验通过后检查用户是否有注册  `InstantiationAwareBeanPostProcessor`。
如果有，使用责任链模式激活这些后置器中的 `postProcessAfterInstantiation` 方法，如果某个后置处理器返回了 false，那么 Spring 就不会执行框架的自动装配逻辑了。

这是在实例化 bean 之后，Spring 属性填充之前执行的钩子方法,  是在 Spring 的自动装配开始之前对该 Bean 实例执行自定义字段注入的回调，也是最后一次机会在自动装配前修改 Bean 的属性值。

例如：
```java
@Component
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {

		if ("userService".equals(beanName)) {
			UserService userService = (UserService) bean;
			userService.test();
		}

		return true;
	}
}
```
上述代码就是对 userService 所实例化出来的对象进行处理。

不过官方的建议是不建议去扩展此后置处理器，而是推荐扩展自 `BeanPostProcessor` 或从 `InstantiationAwareBeanPostProcessorAdapter` 派生。

#### 自动装配

```java
int resolvedAutowireMode = mbd.getResolvedAutowireMode();  
if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {  
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);  
    // Add property values based on autowire by name if applicable.  
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {  
        /*** 根据名称注入 */  
        autowireByName(beanName, mbd, bw, newPvs);  
    }  
    // Add property values based on autowire by type if applicable.  
    if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {  
        /*** 根据类型注入 */  
        autowireByType(beanName, mbd, bw, newPvs);  
    }  
    pvs = newPvs;  
}
```

如果 `bean` 在声明的时候指定了自动注入类型是 `byName` 或 `byType`，则会根据这个规则，对 `bean` 内部排除某些特定的属性，进行 `byName` 或 `byType` 的自动装配，但是大部分情况下不会走这个流程，**byConstructor 这种注入模型在创建对象的时候已经处理过**了。
这里都是对自动注入进行处理，byName 跟 byType 两种注入模型均是依赖 setter 方法。
- byName，根据 setter 方法的名字来查找对应的依赖，例如 setA,那么就是去容器中查找名字为 a 的 Bean；
* byType，根据 setter 方法的参数类型来查找对应的依赖，例如 setXx(A a)，就是去容器中查询类型为 A 的 bean。

#### 处理属性

如果没有显式声明自动装配的方式，那么就会使用到 `InstantiationAwareBeanPostProcessor` 这个后置处理器的 `postProcessProperties` 方法。@Autowired、@Resource、@Value 等注解，就是通过 `InstantiationAwareBeanPostProcessor.postProcessProperties` 扩展点来实现的。
```java
/*** 检查是否有InstantiationAwareBeanPostProcessor */  
boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);  
PropertyDescriptor[] filteredPds = null;  
if (hasInstAwareBpps) {  
    if (pvs == null) {  
        pvs = mbd.getPropertyValues();  
    }  
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
        /**  
         * 这里会执行postProcessProperties方法,处理实例化出来的对象的属性.  
         * 如果是Autowired的InstantiationAwareBeanPostProcessor，就会处理Autowired注解。  
         */  
        PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);  
        if (pvsToUse == null) {  
            if (filteredPds == null) {  
                /*** 得到需要进行依赖检查的属性的集合 */  
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);  
            }  
            /**  
             * 对所有需要依赖检查的属性做后置处理，  
             * 这个方法已经过时了，放到这里就是为了兼容老版本  
             */  
            pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);  
            if (pvsToUse == null) {  
                return;  
            }  
        }  
        pvs = pvsToUse;  
    }  
}
```

`AutowiredAnnotationBeanPostProcessor#postProcessProperties` ：
```java
@Override  
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {  
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);  
    try {  
        metadata.inject(bean, beanName, pvs);  
    } catch (BeanCreationException ex) {  
        throw ex;  
    } catch (Throwable ex) {  
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);  
    }  
    return pvs;  
}
```

Spring 会将所有需要被注入的属性/方法封装成一个 InjectedElement，然后放入到 InjectionMetadata 中，而这个 InjectionMetada 是位于后置处理器中的，这是一个策略模式的应用，不同的后置处理器处理不同的注入类型，而在当前这一步，就是遍历这些不同的后置处理器，开始将它们中的 InjectionMetadata 拿出来, 取出一个个 InjectedElement 完成注入。

得益于这个机制，我们甚至可以实现一个自己的自动注入功能，比如：
```java
@Component
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			for (Field field : bean.getClass().getFields()) {
				if (field.isAnnotationPresent(ZhouyuInject.class)) {
					field.setAccessible(true);
					try {
						field.set(bean, "123");
					} catch (IllegalAccessException e) {
						e.printStackTrace();
					}
				}
			}
		}

		return pvs;
	}
}
```


#### 依赖检查

```java
if (needsDepCheck) {  
    if (filteredPds == null) {  
        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);  
    }  
    checkDependencies(beanName, mbd, filteredPds, pvs);  
}
```

这部分，Spring 在最新的版本已经不推荐使用，这里也不做重点讲解了。


#### 属性注入

```java
if (pvs != null) {  
    /*** 属性都获取完，就开始进行注入了 */  
    applyPropertyValues(beanName, mbd, bw, pvs);  
}
```

Spring 就会开始遍历里面的一个个 PropertyValue，通过反射调用 setXXX 方法来完成注入。所以这就很好理解为什么当注入模型为 byName/byType 的时候，Spring 能完成自动注入了。


### 7. initializeBean-初始化

initializeBean()方法依次调用四个方法
1. invokeAwareMethods()
2. applyBeanPostProcessorsBeforeInitialization()
3. invokeInitMethods()
4. applyBeanPostProcessorsAfterInitialization()

#### invokeAwareMethods

```java
if (System.getSecurityManager() != null) {  
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {  
        invokeAwareMethods(beanName, bean);  
        return null;    }, getAccessControlContext());  
} else {  
    invokeAwareMethods(beanName, bean);  
}
```

一开始，调用了 invokeAwareMethods 方法，这个方法很简单，完成了 Aware 接口的激活功能。可以简单的说：
- 如果 Bean 实现了 `BeanNameAware` 接口，则将 beanName 设值进去；
- 如果 Bean 实现了 `BeanClassLoaderAware` 接口，则将 ClassLoader 设值进去；
- 如果 Bean 实现了 `BeanFactoryAware` 接口，则将 beanFactory 设值进去。

#### applyBeanPostProcessorsBeforeInitialization-初始化前

```java
Object wrappedBean = bean;  
if (mbd == null || !mbd.isSynthetic()) {  
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
}
```

实际上调用的方法：
```java
@Override  
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)  
    throws BeansException {  
    Object result = existingBean;  
    for (BeanPostProcessor processor : getBeanPostProcessors()) {  
        /**  
         * 执行初始化前的BeanPostProcessor.  
         * PostConstruct是在这里调用的。  
         */  
        Object current = processor.postProcessBeforeInitialization(result, beanName);  
        if (current == null) {  
            return result;  
        }  
        result = current;  
    }  
    return result;  
}
```

这一步属于 Bean 生命周期中的初始化前，这也是 Spring 提供的一个扩展点：BeanPostProcessor.postProcessBeforeInitialization()，利用初始化前，我们可以对进行了依赖注入的 Bean 进行处理。比如：

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化前");
		}
		return bean;
	}
}
```

#### invokeInitMethods- Bean初始化方法调用

```java
try {  
    invokeInitMethods(beanName, wrappedBean, mbd);  
} catch (Throwable ex) {  
    throw new BeanCreationException(  
        (mbd != null ? mbd.getResourceDescription() : null),  
        beanName, "Invocation of init method failed", ex);  
}
```

这里就是调用初始化方法，继续看源码 invokeInitMethods：
```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)  
    throws Throwable {  
    /*** 判断Bean是否实现了InitializingBean接口 */  
    boolean isInitializingBean = (bean instanceof InitializingBean);  
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {  
        if (logger.isTraceEnabled()) {  
            logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");  
        }  
        if (System.getSecurityManager() != null) {  
            try {  
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {  
                    ((InitializingBean) bean).afterPropertiesSet();  
                    return null;                }, getAccessControlContext());  
            } catch (PrivilegedActionException pae) {  
                throw pae.getException();  
            }  
        } else {  
            /*** 直接调用afterPropertiesSet方法 */  
            ((InitializingBean) bean).afterPropertiesSet();  
        }  
    }  
    if (mbd != null && bean.getClass() != NullBean.class) {  
        String initMethodName = mbd.getInitMethodName();  
        /*** 判断是否指定了init-method() */  
        if (StringUtils.hasLength(initMethodName) &&  
            !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&  
            !mbd.isExternallyManagedInitMethod(initMethodName)) {  
            /*** 利用反射调用init-method()，*/  
            invokeCustomInitMethod(beanName, bean, mbd);  
        }  
    }  
}
```

这里还涉及到一个老生常谈的问题，`init-method` 与 `isInitializingBean` 谁优先执行。上面源码中已经看的很清楚，`invokeCustomInitMethod`  发生在 `isInitializingBean`  之后。

- 实现 InitializingBean 接口是直接调用 afterPropertiesSet 方法，比通过反射调用 init-method 指定的方法效率相对来说要高点。但是 init-method 方式消除了对 Spring 的依赖；
- 如果调用 afterPropertiesSet 方法时出错，则不调用 init-method 指定的方法。

#### applyBeanPostProcessorsAfterInitialization-初始化后

```java
if (mbd == null || !mbd.isSynthetic()) {  
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
}
```


直接看源码 applyBeanPostProcessorsAfterInitialization 方法源码：
```java
@Override  
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)  
    throws BeansException {  
    Object result = existingBean;  
    for (BeanPostProcessor processor : getBeanPostProcessors()) {  
        Object current = processor.postProcessAfterInitialization(result, beanName);  
        if (current == null) {  
            return result;  
        }  
        result = current;  
    }  
    return result;  
}
```

代码比较简单，和之前碰到 BeanPostProcessor 类似，获得所有 `BeanPostProcessor`，循环执行 `BeanPostProcessor` 中的 `postProcessAfterInitialization` 初始化后方法。

如果 `初始化后` 方法返回的结果集为 null，则返回 `result`。如果不为 null，则将当前方法返回的结果暂存起来，继续执行下一个 `BeanPostProcessor` 中的  `postProcessBeforeInitialization` 初始化后方法。

这是 Bean 创建生命周期中的最后一个步骤，可以在这个步骤中，对 Bean 最终进行处理，Spring 中的 **AOP 就是基于初始化后实现**的，**初始化后返回的对象才是最终的 Bean 对象**。

#### 8. 循环依赖的处理

```java
if (earlySingletonExposure) {  
    Object earlySingletonReference = getSingleton(beanName, false);  
    if (earlySingletonReference != null) {  
        if (exposedObject == bean) {  
            exposedObject = earlySingletonReference;  
        } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {  
            String[] dependentBeans = getDependentBeans(beanName);  
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);  
            for (String dependentBean : dependentBeans) {  
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                    actualDependentBeans.add(dependentBean);  
                }  
            }  
            if (!actualDependentBeans.isEmpty()) {  
                throw new BeanCurrentlyInCreationException(beanName,  
                    "Bean with name '" + beanName + "' has been injected into other beans [" +  
                        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +  
                        "] in its raw version as part of a circular reference, but has eventually been " +  
                        "wrapped. This means that said other beans do not use the final version of the " +  
                        "bean. This is often the result of over-eager type matching - consider using " +  
                        "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");  
            }  
        }  
    }  
}
```

这部分代码又是处理循环依赖的，这里先跳过，后续在循环依赖中详细说明。

#### 9. registerDisposableBeanIfNecessary-Bean 销毁

创建 Bean 完成后并不是就直接结束了，我们有时候还需要定义 Bean 销毁的逻辑：
```java
try {  
    registerDisposableBeanIfNecessary(beanName, bean, mbd);  
} catch (BeanDefinitionValidationException ex) {  
    throw new BeanCreationException(  
        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
}
```
这部分内容也不详细介绍了，后续在 Bean 的销毁分析中详细说明。


至此，Bean 的实例化就结束了，接下来就是将这个创建的 Bean 对象返回，一步一步 return 回去即可。

## 结语

本文主要介绍了 Bean 实例化的核心流程源码 createBean，这个方法中有一大半 Bean 的生命周期，值得重点关注。