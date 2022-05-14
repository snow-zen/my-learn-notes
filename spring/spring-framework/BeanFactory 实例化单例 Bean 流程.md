# BeanFactory 实例化单例 Bean 流程
#Java #Spring 

在 Spring 上下文的整个刷新流程中，除了流程中需要的组件被提起实例化外，剩余的普通 Bean 都是通过调用 BeanFactory 的 preInstantiateSingletons 方法完成实例化。

preInstantiateSingletons 方法源码如下：

```java
@Override  
public void preInstantiateSingletons() throws BeansException {   
  
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);  // 1
  
   // Trigger initialization of all non-lazy singleton beans...  
   for (String beanName : beanNames) {  
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);  // 2
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {  
         if (isFactoryBean(beanName)) {  // 3
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
            if (bean instanceof FactoryBean) {  
               FactoryBean<?> factory = (FactoryBean<?>) bean;  
               boolean isEagerInit;  
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {  
                  isEagerInit = AccessController.doPrivileged(  
                        (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,  
                        getAccessControlContext());  
               }  
               else {  
                  isEagerInit = (factory instanceof SmartFactoryBean &&  
                        ((SmartFactoryBean<?>) factory).isEagerInit());  
               }  
               if (isEagerInit) {  
                  getBean(beanName);  
               }  
            }  
         }  
         else {  
            getBean(beanName);  
         }  
      }  
   }  
  
   // Trigger post-initialization callback for all applicable beans...  
   for (String beanName : beanNames) {  
      Object singletonInstance = getSingleton(beanName);  
      if (singletonInstance instanceof SmartInitializingSingleton) { // 4  
         StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")  
               .tag("beanName", beanName);  
         SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;  
         if (System.getSecurityManager() != null) {  
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {  
               smartSingleton.afterSingletonsInstantiated();  
               return null;            }, getAccessControlContext());  
         }  
         else {  
            smartSingleton.afterSingletonsInstantiated();  
         }  
         smartInitialize.end();  
      }  
   }  
}
```

1. 拷贝所有的 Bean 定义名称以进行循环处理。
2. 根据 Bean 定义名称获取到对应的 Bean 定义信息。
3. 检查 Bean 定义名称对应的 Bean 或者 Bean 定义信息中是否表示 Bean 为 FactoryBean 类型。如果是 FactoryBean 类型，则根据请求判断是否需要提早初始化。
4. 所有 Bean 完成实例化后，根据 Bean 定义名称获取到对应的单例 Bean 后，检查 Bean 是否为 SmartInitializingSingleton 类型，匹配则回调对应的 afterSingletonsInstantiated 方法。

其中的 getBean 方法作为整个实例化过程的基础方法，用于对单个 Bean 进行实例化。

## getBean

该方法被委托给 doGetBean 方法去完成。进入方法首先就是通过名称尝试获取是否存在对应的单例 Bean：

```java
// doGetBean 方法中
Object sharedInstance = getSingleton(beanName);
```

该方法查询 BeanFactory 中当前已实例化的 Bean 中是否存在匹配的，存在则无须再进行实例化。在检查到不存在时，随后检查当前 BeanFactory 是否存在父级 BeanFactory，如果存在则从父级 BeanFactory 尝试实例化对象：

```java
// doGetBean 方法中
BeanFactory parentBeanFactory = getParentBeanFactory();  
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {  
   // Not found -> check parent.  
   String nameToLookup = originalBeanName(name);  
   if (parentBeanFactory instanceof AbstractBeanFactory) {  
      return ((AbstractBeanFactory) parentBeanFactory).doGetBean(  
            nameToLookup, requiredType, args, typeCheckOnly);  
   }  
   else if (args != null) {  
      // Delegation to parent with explicit args.  
      return (T) parentBeanFactory.getBean(nameToLookup, args);  
   }  
   else if (requiredType != null) {  
      // No args -> delegate to standard getBean method.  
      return parentBeanFactory.getBean(nameToLookup, requiredType);  
   }  
   else {  
      return (T) parentBeanFactory.getBean(nameToLookup);  
   }  
}
```

当从父级 BeanFactory 中无法实例化 Bean 时，则继续由当前 BeanFactory 进行实例化。在实例化前，首先将 Bean 标记为已创建：

```java
// doGetBean 方法中
if (!typeCheckOnly) {  
   markBeanAsCreated(beanName);  
}
```

`typeCheckOnly` 属性是在调用 doGetBean 方法时传入的，getBean 方法默认传入值为 false。随后获取对应的 Bean 定义信息并且检查相关信息：

```java
// doGetBean 方法中
RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);  
checkMergedBeanDefinition(mbd, beanName, args);
```

检查方法主要检查对应的 Bean 是否为抽象类。随后获取 Bean 所依赖的组件并在实例化 Bean 前先实例化所依赖的组件：

```java
// doGetBean 方法中
String[] dependsOn = mbd.getDependsOn();  
if (dependsOn != null) {  
   for (String dep : dependsOn) {  
      if (isDependent(beanName, dep)) {  
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
               "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");  
      }  
      registerDependentBean(dep, beanName);  
      try {  
         getBean(dep);  
      }  
      catch (NoSuchBeanDefinitionException ex) {  
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
               "'" + beanName + "' depends on missing bean '" + dep + "'", ex);  
      }  
   }  
}
```

且在实例化依赖组件时，还进行标记以防止循环依赖的情况出现。当依赖组件完成实例化后，则可以开始实例化当前 Bean。实例化根据 Bean 是 Singleton，还是 Prototype 或者是其他的。这里以常用的 Singleton 为例：

```java
// doGetBean 方法中
if (mbd.isSingleton()) {  
   sharedInstance = getSingleton(beanName, () -> {  
      try {  
         return createBean(beanName, mbd, args);  
      }  
      catch (BeansException ex) {  
         destroySingleton(beanName);  
         throw ex;  
      }  
   });  
   beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);  
}
```

不同于 doGetBean 方法开头调用的 getSingleton 方法，该方法会额外传入一个 Lambda 函数对象。方法首先从本地获取对应的 Bean，如果不存在则先将 Bean 标记为已创建的状态，之后再调用 Lambda 函数对象创建对应的实例。而该 Lambda 函数对象则是调用 createBean 方法创建对应的对象。

## createBean

创建 Bean 首先就是解析对应的 Bean 定义信息，其中第一个需要确认的是 Bean 的类型信息。如果可以解析到类型信息则将其设置到 Bean 定义信息中：

```java
// createBean 方法内
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);  
if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {  
   mbdToUse = new RootBeanDefinition(mbd);  
   mbdToUse.setBeanClass(resolvedClass);  
}
```

然后对覆盖方法进行处理：

```java
// createBean 方法内
try {  
   mbdToUse.prepareMethodOverrides();  
}  
catch (BeanDefinitionValidationException ex) {  
   throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),  
         beanName, "Validation of method overrides failed", ex);  
}
```

在开始实例化 Bean 之前先执行 InstantiationAwareBeanPostProcessor 后置处理的实例化前回调让其有机会直接返回一个代理对象：

```java
// createBean 方法内
try {  
   // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.  
   Object bean = resolveBeforeInstantiation(beanName, mbdToUse);  
   if (bean != null) {  
      return bean;  
   }  
}  
catch (Throwable ex) {  
   throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,  
         "BeanPostProcessor before instantiation of bean failed", ex);  
}
```

完成实例化前回调后，则调用 doCreateBean 方法进入真正实例化的阶段。

## doCreateBean

创建时，首先考虑从未完成实例化的 FactoryBean 中尝试获取对应的 Bean：

```java
// doCreateBean 方法中
BeanWrapper instanceWrapper = null;  
if (mbd.isSingleton()) {  
   instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);  
}
```

当不存在才会开始调用 createBeanInstance 方法进行实例化：

```java
// doCreateBean 方法中
if (instanceWrapper == null) {  
   instanceWrapper = createBeanInstance(beanName, mbd, args);  
}
```

在 createBeanInstance 方法中，在获取到对应的 Class 对象后进一步需要确定的是实例化所使用的构造方法，这里调用 determineConstructorsFromBeanPostProcessors 方法进行查找：

```java
@Nullable  
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)  
      throws BeansException {  
  
   if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {  
      for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {  
         Constructor<?>[] ctors = bp.determineCandidateConstructors(beanClass, beanName);  
         if (ctors != null) {  
            return ctors;  
         }  
      }  
   }  
   return null;  
}
```

当 BeanFactory 中存在 SmartInstantiationAwareBeanPostProcessor 后置处理器时，则可返回指定的构造方法。如果不存在时则返回 null，当上层方法无法接收到指定的构造方法时，则使用默认的无参构造方法进行实例化。

获得刚刚实例化完成的 Bean 对象后，随后就进行一次回调，允许所有的 MergedBeanDefinitionPostProcessor 后置处理器修改 Bean 定义信息：

```java
// doCreateBean 方法中
synchronized (mbd.postProcessingLock) {  
   if (!mbd.postProcessed) {  
      try {  
         applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
      }  
      catch (Throwable ex) {  
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
               "Post-processing of merged bean definition failed", ex);  
      }  
      mbd.postProcessed = true;  
   }  
}
```

然后，当 BeanFactory 允许循环引用时，可立刻将 Bean 的引用保持到 BeanFactory 中以解决循环引用：

```java
// doCreateBean 方法中
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

这里有个注意点，在加入 BeanFactory 中所使用的是一个 Lambda 函数对象，其中调用的是 getEarlyBeanReference 方法。该方法会回调所有 SmartInstantiationAwareBeanPostProcessor 后置处理器方法来对返回对象进行处理：

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {  
   Object exposedObject = bean;  
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
      for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {  
         exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);  
      }  
   }  
   return exposedObject;  
}
```

完成上述动作后，则进行 Bean 的初始化阶段：

```java
// doCreateBean 方法中
Object exposedObject = bean;  
try {  
   populateBean(beanName, mbd, instanceWrapper);  
   exposedObject = initializeBean(beanName, exposedObject, mbd);  
}  
catch (Throwable ex) {  
   if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {  
      throw (BeanCreationException) ex;  
   }  
   else {  
      throw new BeanCreationException(  
            mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);  
   }  
}
```

初始化阶段首先调用的就是 populateBean 方法，进入方法后首先是 InstantiationAwareBeanPostProcessor 后置处理器的实例化后回调：

```java
// populateBean 方法中
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
   for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
      if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {  
         return;  
      }  
   }  
}
```

之后是 InstantiationAwareBeanPostProcessor 后置处理器的属性后置处理回调：

```java
// populateBean 方法中
for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
   PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);  
   if (pvsToUse == null) {  
      if (filteredPds == null) {  
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);  
      }  
      pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);  
      if (pvsToUse == null) {  
         return;  
      }  
   }  
   pvs = pvsToUse;  
}
```

populateBean 方法完成前置动作后，则调用 initializeBean 方法完成初始化。首先是调用指定 Aware 接口相关方法：

```java
// initializeBean 方法中
invokeAwareMethods(beanName, bean);
```

该方法主要处理以下几种 Aware 接口：

+ BeanNameAware
+ BeanClassLoaderAware
+ BeanFactoryAware

随后通过 applyBeanPostProcessorsBeforeInitialization 方法调用 BeanPostProcessor 后置处理器的初始化前回调：

```java
@Override  
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)  
      throws BeansException {  
  
   Object result = existingBean;  
   for (BeanPostProcessor processor : getBeanPostProcessors()) {  
      Object current = processor.postProcessBeforeInitialization(result, beanName);  
      if (current == null) {  
         return result;  
      }  
      result = current;  
   }  
   return result;  
}
```

紧接着调用 invokeInitMethods 方法完成初始化工作，当 Bean 实现了 InitializingBean 或者指定了初始化方法时，则完成指定的初始化逻辑。

最后，通过调用 applyBeanPostProcessorsAfterInitialization 方法完成 BeanPostProcessor 后置处理器的初始化后回调：

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










