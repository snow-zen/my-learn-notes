# Spring AOP 工作原理和流程
#Java #Spring 

AOP 指面向切面编程，通过在运行时对指定元素进行功能增强。Spring AOP 功能主要通过 JDK 动态代理和 CGLIB 两种方式来实现。

通过注解方式使用 Spring AOP 时，需要在配置类中添加 `@EnableAspectJAutoProxy` 注解开启 AOP 功能。

## @EnableAspectJAutoProxy 注解

`@EnableAspectJAutoProxy` 注解主要有三个作用，源码如下：

```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Import(AspectJAutoProxyRegistrar.class)  
public @interface EnableAspectJAutoProxy {  
  
    boolean proxyTargetClass() default false;  
  
    boolean exposeProxy() default false;  
  
}
```

首先，该注解定义使用 `@Import(AspectJAutoProxyRegistrar.class)` 进行标注，也就意味这它通过 AspectJAutoProxyRegistrar 类进行 Bean 的注入。

随后，注解包含两个属性：

+ `proxyTargetClass`：表示是否使用 CGLIB 创建代理类，默认为 false。
+ `exposeProxy`：表示 AOP 框架是否需要将被代理对象发布到 ThreadLocal 中去，以便通过 AopContext 进行获取，默认为 false。

## AspectJAutoProxyRegistrar 注入 Bean

AspectJAutoProxyRegistrar 继承了 ImportBeanDefinitionRegistrar 接口，即通过实现 registerBeanDefinitions 方法向 BeanDefinitionRegistry 对象注册 Bean 的定义信息即可。

AspectJAutoProxyRegistrar 类源码也非常简单：

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {  
  
   @Override  
   public void registerBeanDefinitions(  
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {  
  
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);  // 1
  
      AnnotationAttributes enableAspectJAutoProxy =  
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);  
      if (enableAspectJAutoProxy != null) {  
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {  
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry); // 2  
         }  
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {  
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);  // 3
         }  
      }  
   }  
  
}
```

在注释 1 处通过调用 registerAspectJAnnotationAutoProxyCreatorIfNecessary 方法向 `registry` 对象注册一个特殊的 Bean 定义信息。在该方法内部将实现委托给另一个 registerOrEscalateApcAsRequired 方法并额外传入 AnnotationAwareAspectJAutoProxyCreator 类信息：

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(  
      BeanDefinitionRegistry registry, @Nullable Object source) {  
  
   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);  
}
```

在 registerOrEscalateApcAsRequired 方法中才是真正注册 AnnotationAwareAspectJAutoProxyCreator 类 Bean 的定义信息：

```java
private static BeanDefinition registerOrEscalateApcAsRequired(  
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {   
  
   // ...  
  
   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);  
   beanDefinition.setSource(source);  
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);  
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);  
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);  
   return beanDefinition;  
}
```

之后，AspectJAutoProxyRegistrar 类在注释 2 和 3 处通过读取 `@EnableAspectJAutoProxy` 注解的两个属性对上面所注册的特殊 Bean 定义信息设置属性：

```java
public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {  
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {  
      BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);  
      definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);  
   }  
}

public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {  
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {  
      BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);  
      definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);  
   }  
}
```

通过这两个方法源码可以看到在调用方法后，它在 Bean 定义信息中分别将 `proxyTargetClass` 属性和 `exposeProxy` 属性设置为 true。

## AnnotationAwareAspectJAutoProxyCreator

AnnotationAwareAspectJAutoProxyCreator 的 Bean 定义信息在被注册到容器中后，被 BeanFactory 实例化成具体的对象。

AnnotationAwareAspectJAutoProxyCreator 的部分 UML 结构如下：

![AnnotationAwareAspectJAutoProxyCreator UML 结构](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/AnnotationAwareAspectJAutoProxyCreator.png)

可以知道该类属于 SmartInstantiationAwareBeanPostProcessor 这一组件，此处我们主要关心三处回调。

**postProcessBeforeInstantiation**

该回调方法在 InstantiationAwareBeanPostProcessor 中定义，在 Bean 实例化前回调。该回调实现如下：

```java
@Override  
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {  
   Object cacheKey = getCacheKey(beanClass, beanName);  
  
   if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {  // 1
      if (this.advisedBeans.containsKey(cacheKey)) {  
         return null;  
      }  
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {  
         this.advisedBeans.put(cacheKey, Boolean.FALSE);  
         return null;      }  
   }  
  
   TargetSource targetSource = getCustomTargetSource(beanClass, beanName);  // 2
   if (targetSource != null) {  
      if (StringUtils.hasLength(beanName)) {  
         this.targetSourcedBeans.add(beanName);  
      }  
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);  
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);  // 3
      this.proxyTypes.put(cacheKey, proxy.getClass());  
      return proxy;  
   }  
  
   return null;  
}
```

1. 当 beanName 为空或者不是 TargetSource 相关 Bean 的名称时，进行以下检查：
	+ `advisedBeans` 属性中是否存在对应 key 的条目，存在则说明该 Bean 已经创建且被代理完成，则可直接跳过后续步骤。
    + 当 beanClass 类对象为 Advice、Pointcut、Advisor、AopInfrastructureBean 或者是原始实例时，则标记为 False 用于跳过代理过程。
2. 当存在自定义的 TargetSource 时，则可当即创建对应的代理对象，并返回代理对象，此时后续的实例化动作将不再执行。[^1]

[^1]: 该机制非常适合实现延迟创建对象等功能。

**postProcessAfterInitialization**

该回调在 InstantiationAwareBeanPostProcessor 中定义，在 Bean 实例化后回调。该回调实现如下：

```java
@Override  
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {  
   if (bean != null) {  
      Object cacheKey = getCacheKey(bean.getClass(), beanName);  
      if (this.earlyProxyReferences.remove(cacheKey) != bean) { // 1 
         return wrapIfNecessary(bean, beanName, cacheKey);  // 2
      }  
   }  
   return bean;  
}
```

1. 该代码主要是删除早期的代理对象缓存，用于解决循环引用问题。
2. 当早期的代理对象引用与当前 bean 不同时，则调用 wrapIfNecessary 方法确认是否需要包装代理。

wrapIfNecessary 方法是主要的创建代理对象方法，核心实现如下：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  
   // ...  
  
   // Create proxy if we have advice.  
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null); // 1 
   if (specificInterceptors != DO_NOT_PROXY) {  
      this.advisedBeans.put(cacheKey, Boolean.TRUE);  
      Object proxy = createProxy(  
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));  // 2
      this.proxyTypes.put(cacheKey, proxy.getClass());  
      return proxy;  
   }  
  
   this.advisedBeans.put(cacheKey, Boolean.FALSE);  
   return bean;  
}
```

1. 尝试匹配到相关的切面类对象。
2. 将切面类对象和 Bean 实例通过 createProxy 方法编织成代理对象，且直接使用 SingletonTargetSource 对象。

createProxy 方法首先创建对应的代理工作，然后再进行代理：

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,  
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {  
  
   // ...  
  
   ProxyFactory proxyFactory = new ProxyFactory();  
   proxyFactory.copyFrom(this);  
  
   if (proxyFactory.isProxyTargetClass()) {  // 1
      // Explicit handling of JDK proxy targets and lambdas (for introduction advice scenarios)  
      if (Proxy.isProxyClass(beanClass) || ClassUtils.isLambdaClass(beanClass)) {  
         // Must allow for introductions; can't just set interfaces to the proxy's interfaces only.  
         for (Class<?> ifc : beanClass.getInterfaces()) {  
            proxyFactory.addInterface(ifc);  
         }  
      }  
   }  
   else {  
      // No proxyTargetClass flag enforced, let's apply our default checks...  
      if (shouldProxyTargetClass(beanClass, beanName)) {  
         proxyFactory.setProxyTargetClass(true);  
      }  
      else {  
         evaluateProxyInterfaces(beanClass, proxyFactory);  
      }  
   }  
  
   // ...
  
   // Use original ClassLoader if bean class not locally loaded in overriding class loader  
   ClassLoader classLoader = getProxyClassLoader();  
   if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {  // 2
      classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();  
   }  
   return proxyFactory.getProxy(classLoader);  // 3
}
```

1. 通过代理工厂的 isProxyTargetClass 方法确认是否需要确认接口信息。该方法的实现使用的是在 `@EnableAspectJAutoProxy` 中的 `proxyTargetClass` 属性。
2. 获取指定 ClassLoader 用于加载类型信息。
3. 调用 getProxy 方法创建代理。

getProxy 方法是一个委托方法，实际真正调用的是 createAopProxy 方法所返回 AopProxy 对象的 getProxy 方法：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {  
   return createAopProxy().getProxy(classLoader);  
}
```

createAopProxy 方法源码如下：

```java
protected final synchronized AopProxy createAopProxy() {  
   if (!this.active) {  
      activate();  // 1
   }  
   return getAopProxyFactory().createAopProxy(this);  // 2
}
```

1. 在首次未激活的状态下，它会调用 activate 方法。方法内部回调所有 AdvisedSupportListener 对象的 activated 方法。
2. 在调用 getAopProxyFactory 方法获取工厂对象后，调用 createAopProxy 方法。

createAopProxy 方法则用于确认代理的策略：

```java
@Override  
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {  
   if (!NativeDetector.inNativeImage() &&  
         (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {  
      Class<?> targetClass = config.getTargetClass();  
      if (targetClass == null) {  
         throw new AopConfigException("TargetSource cannot determine target class: " +  
               "Either an interface or a target is required for proxy creation.");  
      }  
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {  
         return new JdkDynamicAopProxy(config);  
      }  
      return new ObjenesisCglibAopProxy(config);  
   }  
   else {  
      return new JdkDynamicAopProxy(config);  
   }  
}
```

可以看到实现根据配置信息选择从 CGLIB 和 JDK 动态代理中选择一种方式的对象，随后调用其 getProxy 方法。






