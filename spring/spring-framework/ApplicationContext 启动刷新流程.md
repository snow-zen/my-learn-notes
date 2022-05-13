# ApplicationContext 启动刷新流程
#Java #Spring 

ApplicationContext 是 Spring 框架的核心之一，用于管理整个 Spring 应用程序的上下文和生命周期。而 ApplicationContext 整个生命周期的重点在于刷新容器，以配置 BeanFactory 和实例化所有的 Bean 以及相关回调。

本次使用 Spring Boot Web 应用程序中所使用的具体实现 AnnotationConfigServletWebServerApplicationContext 作为具体分析实现类，探究 AbstractApplicationContext 中的 refresh 方法。

## 准备阶段

准备阶段主要完成一些前置动作，例如：状态的重置、BeanFactory 的配置。

### prepareRefresh

该阶段主要对部分属性的状态进行设置，例如将 ApplicationContext 设置为活跃状态：

```java
// prepareRefresh 方法内

this.closed.set(false);  
this.active.set(true);
```

之后调用 initPropertySources 方法初始化上下文环境中的占位符属性，因此该方法的实现被委托给了 ConfigurableEnvironment 对象进行初始化：

```java
@Override  
protected void initPropertySources() {  
   ConfigurableEnvironment env = getEnvironment();  
   if (env instanceof ConfigurableWebEnvironment) {  
      ((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, null);  
   }  
}
```

这里需要注意一点的是，只有类型为 ConfigurableWebEnvironment 的环境对象才会去初始化属性，这表示该动作只在 Web 环境中进行。而在环境对象的 initPropertySources 方法实现也被委托给 WebApplicationContextUtils.initServletPropertySources 静态方法。其方法实现如下：

```java
public static void initServletPropertySources(MutablePropertySources sources,  
      @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {  
    
   String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME; // 1  
   if (servletContext != null && sources.get(name) instanceof StubPropertySource) {  
      sources.replace(name, new ServletContextPropertySource(name, servletContext));  
   }  
   name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME; // 2 
   if (servletConfig != null && sources.get(name) instanceof StubPropertySource) {  
      sources.replace(name, new ServletConfigPropertySource(name, servletConfig));  
   }  
}
```

1. 当环境对象中 `servletContext` 属性不为 null 时，设置 servletContextInitParams 属性对象。
2. 当环境对象中 `servletConfig` 属性不为 null 时，设置 servletConfigInitParams 属性对象。

在完成属性对象初始化后，则通过调用环境对象的 validateRequiredProperties 方法验证所有必需属性：

```java
@Override  
public void validateRequiredProperties() throws MissingRequiredPropertiesException {  
   this.propertyResolver.validateRequiredProperties();  
}
```

而在 validateRequiredProperties 方法具体实现中则是检查 requiredProperties 属性中所指定的属性是否存在，当至少有一个属性不存在时，则抛出 MissingRequiredPropertiesException 异常：

```java
@Override  
public void validateRequiredProperties() {  
   MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();  
   for (String key : this.requiredProperties) {  
      if (this.getProperty(key) == null) {  
         ex.addMissingRequiredProperty(key);  
      }  
   }  
   if (!ex.getMissingRequiredProperties().isEmpty()) {  
      throw ex;  
   }  
}
```

验证完必需属性后，则对早期保存的 ApplicationListener 监听器进行处理，并且清空所有早期的事件：

```java
// prepareRefresh 方法内

if (this.earlyApplicationListeners == null) {  // 1
   this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);  
}  
else {  // 2 
   this.applicationListeners.clear();  
   this.applicationListeners.addAll(this.earlyApplicationListeners);  
}  
    
this.earlyApplicationEvents = new LinkedHashSet<>();
```

1. 当早期监听器列表为空时，则将当前监听器列表复制到其中。
2. 当早期监听器列表不为空时，则清空当前监听器并复制早期监听器列表。

## obtainFreshBeanFactory

该阶段主要是获取上下文中的 BeanFactory 对象，并且在返回对象前由子类完成一些重置动作。

```java
// refresh 方法内

ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

## prepareBeanFactory

在使用 BeanFactory 之前通过 prepareBeanFactory 方法对其进行前置配置。首先配置的就是 BeanFactory 在加载 Bean 的类对象时所使用到的 ClassLoader 对象：

```java
beanFactory.setBeanClassLoader(getClassLoader());
```

随后根据从类路径下的 spring.properties 文件中读取到的 `spring.spel.ignore` 值作为 `shouldIgnoreSpel` 字段值来决定是否在 BeanFactory 中加入 BeanExpressionResolver：

```java
// prepareBeanFactory 方法内
if (!shouldIgnoreSpel) {  
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));  
}
```

后面紧跟着的是设置 PropertyEditorRegistrar：

```java
// prepareBeanFactory 方法内
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

之后在 BeanFactory 中添加一个 ApplicationContextAwareProcessor 后置处理器用于在 Bean 初始化前时回调对应 Aware 方法，同时在 BeanFactory 中忽略注入该后置处理器所能解析的 Aware 接口实例：

```java
// prepareBeanFactory 方法内  
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));  
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);  
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);  
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);  
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);  
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);  
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);  
beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);
```

对于一些特殊类型的 Bean 的注入，则在此设置为固定实例：

```java
// prepareBeanFactory 方法内  
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);  
beanFactory.registerResolvableDependency(ResourceLoader.class, this);  
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);  
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

然后又向 BeanFactory 中添加 ApplicationListenerDetector 后置处理器，用于将实现了 ApplicationListener 接口的 Bean 自动加入到监听器列表：

```java
// prepareBeanFactory 方法内  
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
```

再接着做了一次判断：如果 BeanFactory 对象中存在名称为 loadTimeWeaver 的 Bean 时，则向 BeanFactory 中添加 LoadTimeWeaverAwareProcessor 后置处理器以及设置临时 ClassLoader 为 ContextTypeMatchClassLoader：

```java
// prepareBeanFactory 方法内  
if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {  
   beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));  
   // Set a temporary ClassLoader for type matching.  
   beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));  
}
```

最后向 BeanFactory 对象添加指定名称的单例 Bean：

```java
// prepareBeanFactory 方法内  
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {  
   beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());  
}  
if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {  
   beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());  
}  
if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {  
   beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());  
}  
if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {  
   beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());  
}
```

至此，准备工作全部完成。

## postProcessBeanFactory

该方法是一个模版方法，主要给子类去实现，在 BeanFacoty 准备完毕后进行的回调：

```java
// refresh 方法内
postProcessBeanFactory(beanFactory);
```

## invokeBeanFactoryPostProcessors

在调用 invokeBeanFactoryPostProcessors 方法是有个注意点，传入的 beanFactoryPostProcessors 参数是从 ApplicationContext 中获取的，这是额外的一部分后置处理器，用于上下文刷新时使用：

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {  
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());  // 1
  
  if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {  // 2
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));  
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));  
   }  
}
```

1. 回调 BeanFactoryPostProcessor 后置处理器的工作委托给了静态方法完成。
2. 此处与 prepareBeanFactory 方法实现中所做的逻辑相似，但此次是应对通过 @Bean 注解从 ConfigurationClassPostProcessor 注入时的情况。

其中 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors 静态方法大致逻辑如下：

```java
public static void invokeBeanFactoryPostProcessors(  
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    if (beanFactory instanceof BeanDefinitionRegistry) {
		// 回调 beanFactoryPostProcessors 中的 BeanDefinitionRegistryPostProcessor 后置处理器
		// 回调 beanFactory 中的 BeanDefinitionRegistryPostProcessor 后置处理器
	} else {
		// 回调 beanFactory 中的 BeanDefinitionRegistryPostProcessor 后置处理器
	}
	
    // 回调 beanFactory 中剩余 BeanFactoryPostProcessor 后置处理器
}
```

在处理从 beanFactory 中获取到的 Bean 时，根据实现了 PriorityOrdered、Ordered 和没有实现相关接口的顺序回调。

### registerBeanPostProcessors

调用 registerBeanPostProcessors 方法可以将所有 BeanPostProcess 后置处理器实例化并添加到 BeanFactory 的后置处理器列表中。所有的 BeanPostProcess 后置处理器同样也是根据实现了 PriorityOrdered、Ordered 和没有实现相关接口的顺序添加进 BeanFactory 中。

在所有的 BeanPostProcess 后置处理器完成添加后，程序又将所有类型为 MergedBeanDefinitionPostProcessor 的后置处理器又重新添加到 BeanFactory 中，放在列表尾部。

最后，程序将 ApplicationListenerDetector 后置处理器重新注册到 BeanFactory 中，放在队列的尾部，以适应代理类的情况。

## initMessageSource

调用 initMessageSource 方法可以在 BeanFactory 查询名称为 messageSource 且类型为 MessageSource 类型的 Bean 并进行设置。

当 BeanFactory 中存在符合条件的 Bean 且类型为 HierarchicalMessageSource 子类型时，则将其设置为上下文的 `messageSource` 属性，并额外设置为父 MessageSource 类型。当 BeanFactory 中不存在时，则设置为默认的 DelegatingMessageSource 类型对象：

```java
protected void initMessageSource() {  
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {  
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);  
      // Make MessageSource aware of parent MessageSource.  
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {  
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;  
         if (hms.getParentMessageSource() == null) {  
            hms.setParentMessageSource(getInternalParentMessageSource());
         }  
      }  
   }  
   else {  
      // Use empty MessageSource to be able to accept getMessage calls.  
      DelegatingMessageSource dms = new DelegatingMessageSource();  
      dms.setParentMessageSource(getInternalParentMessageSource());  
      this.messageSource = dms;  
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);  
   }  
}
```

## initApplicationEventMulticaster

调用 initApplicationEventMulticaster 方法用于在 BeanFactory 设置名称为 applicationEventMulticaster 且类型为 ApplicationEventMulticaster 的事件多播器。

当 BeanFactory 中存在符合条件的 Bean 时，则将其设置为上下文的 `applicationEventMulticaster` 属性。否则，设置为默认的 SimpleApplicationEventMulticaster 类型对象：

```java
protected void initApplicationEventMulticaster() {  
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {  
      this.applicationEventMulticaster =  
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);   
   }  
   else {  
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);  
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);    
   }  
}
```

## onRefresh

在 AbstractApplicationContext 中 onRefresh 方法是一个模版方法，由子类进行重写。以 AnnotationConfigServletWebServerApplicationContext 为例，该子类在此完成两个动作：

```java
@Override  
protected void onRefresh() {  
   super.onRefresh();  // 1
   try {  
      createWebServer();  // 2
   }  
   catch (Throwable ex) {  
      throw new ApplicationContextException("Unable to start web server", ex);  
   }  
}
```

1. 初始化 themeSource 属性。
2. 创建 Web 服务器。

## registerListeners

获取所有的事件监听器并将其注册到 `applicationEventMulticaster` 中，首先注册的是上下文中静态指定的监听器：

```java
// registerListeners 方法内 
for (ApplicationListener<?> listener : getApplicationListeners()) {  
   getApplicationEventMulticaster().addApplicationListener(listener);  
}
```

随后将 BeanFactory 中所有类型为 ApplicationListener 的 Bean 注册到上下文中：

```java
// registerListeners 方法内 
String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);  
for (String listenerBeanName : listenerBeanNames) {  
   getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);  
}
```

最后将早期所积累的事件在此刻全部发布给监听器：

```java
// registerListeners 方法内 
Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;  
this.earlyApplicationEvents = null;  
if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {  
   for (ApplicationEvent earlyEvent : earlyEventsToProcess) {  
      getApplicationEventMulticaster().multicastEvent(earlyEvent);  
   }  
}
```

## finishBeanFactoryInitialization 

该方法用于完成 BeanFactory 的最后初始化工作，其中就包括剩余 Bean 的实例化。首先，为其设置 ConversionService 用于将属性值转换为对于的 Java 类型：

```java
// finishBeanFactoryInitialization 方法内
if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&  
      beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {  
   beanFactory.setConversionService(  
         beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));  
}
```

随后，当 BeanFactory 中不存在内置 value 解析器，设置一个默认的：

```java
// finishBeanFactoryInitialization 方法内
if (!beanFactory.hasEmbeddedValueResolver()) {  
   beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));  
}
```

提早实例化所有的 LoadTimeWeaverAware 类型实例，以便尽早注册它们的转换器。同时将 BeanFactory 中的临时 ClassLoader 设置为 null：

```java
// finishBeanFactoryInitialization 方法内
String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);  
for (String weaverAwareName : weaverAwareNames) {  
   getBean(weaverAwareName);  
}  
   
beanFactory.setTempClassLoader(null);
```

最后，将 BeanFactory 冻结配置并开始实例化剩余 Bean：

```java
// finishBeanFactoryInitialization 方法内
beanFactory.freezeConfiguration();  
  
beanFactory.preInstantiateSingletons();
```

## finishRefresh

至此，已经到了刷新的最后阶段。在该方法中首先做的是清楚所有的资源缓存：

```java
// finishRefresh 方法内
clearResourceCaches();
```

紧跟着就是调用 initLifecycleProcessor 方法初始化名称为 lifecycleProcessor 且类型为 LifecycleProcessor 的 Bean，并将其设置为上下文的 `lifecycleProcessor` 属性：

```java
protected void initLifecycleProcessor() {  
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
   if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {  
      this.lifecycleProcessor =  
            beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);  
   }  
   else {  
      DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();  
      defaultProcessor.setBeanFactory(beanFactory);  
      this.lifecycleProcessor = defaultProcessor;  
      beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);   
   }  
}
```

在初始化完成 LifecycleProcessor 后立即调用它的 onRefresh 方法：

```java
getLifecycleProcessor().onRefresh();
```

最后在上下文中发布 ContextRefreshedEvent 事件，表示上下文已刷新：

```java
publishEvent(new ContextRefreshedEvent(this));
```









