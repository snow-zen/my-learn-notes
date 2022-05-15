# Spring Boot 启动流程
#Java #Spring 

Spring Boot 程序的启动是从调用 SpringApplication 的 run 方法开始的，run 方法的源码如下：

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {  
   return new SpringApplication(primarySources).run(args);  
}
```

可以看到该实现分为两个阶段进行：创建 SpringApplication 实例以及调用实例的 run 方法。

## 创建 SpringApplication 实例

在调用 SpringApplication 构造方法进行实例化时，最终委托给一个参数更全面的构造方法实现：

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {  
   this.resourceLoader = resourceLoader;  
   Assert.notNull(primarySources, "PrimarySources must not be null");  
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));  
   this.webApplicationType = WebApplicationType.deduceFromClasspath();  // 1
   this.bootstrapRegistryInitializers = new ArrayList<>(  
         getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  // 2
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class)); // 3  
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));  // 4
   this.mainApplicationClass = deduceMainApplicationClass(); // 5 
}
```

1. 选择当前 Web 应用程序的类型，该方法根据是否存在特定类来进行判断的，一共有三种类型：NONE、SERVLET、REACTIVE。
2. 通过调用 getSpringFactoriesInstances 方法获取到 BootstrapRegistryInitializer 类型实例并设置到 bootstrapRegistryInitializers 属性中。
3. 通过调用 getSpringFactoriesInstances 方法获取到 ApplicationContextInitializer 类型实例并设置到 initializers 属性中。
4. 通过调用 getSpringFactoriesInstances 方法获取到 ApplicationListener 类型实例并设置到 listeners 属性中。
5. 调用 deduceMainApplicationClass 方法寻找到程序的主启动类，这里利用异常堆栈的特性来获取方法名为 main 的栈，从而确定主启动类。

频繁使用的 getSpringFactoriesInstances 方法是 Spring Boot 的一个基础方法。它实际上是调用 SpringFactoriesLoader 类的 loadFactoryNames 静态方法，从 `META-INF/spring.factories` 文件中读取对于接口的实现类全类名来进行加载的。

## 调用 run 方法

进入方法后，首先是调用 configureHeadlessProperty 方法配置属性，它将 `java.awt.headless` 系统属性默认设置为 true，表示程序可以在没有键盘、鼠标和显示器的情况下运行：

```java
private void configureHeadlessProperty() {  
   System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,  
         System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));  
}
```

然后调用 getRunListeners 方法从 `spring.factories` 文件中获取到所有指定 SpringApplicationRunListener 接口实现类并进行实例化且使用委托类进行包装：

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {  
   Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };  
   return new SpringApplicationRunListeners(logger,  
         getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args),  
         this.applicationStartup);  
}
```

得到相关的监听器之后，则发布 starting 事件回调：

```java
// run 方法内
listeners.starting(bootstrapContext, this.mainApplicationClass);
```

### 准备环境对象

之后通过调用 prepareEnvironment 方法准备环境对象。方法内先是调用 getOrCreateEnvironment 方法获取或者创建对应类型对象：

```java
private ConfigurableEnvironment getOrCreateEnvironment() {  
   if (this.environment != null) {  
      return this.environment;  
   }  
   switch (this.webApplicationType) {  
   case SERVLET:  
      return new ApplicationServletEnvironment();  
   case REACTIVE:  
      return new ApplicationReactiveWebEnvironment();  
   default:  
      return new ApplicationEnvironment();  
   }  
}
```

以 Web 程序为例，这里所返回的是 ApplicationServletEnvironment 类型对象。并且在随后的 configureEnvironment 方法内对环境对象进行配置：

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {  
   if (this.addConversionService) {  
      environment.setConversionService(new ApplicationConversionService());  // 1
   }  
   configurePropertySources(environment, args);  // 2
   configureProfiles(environment, args);  // 3
}
```

1. 配置转换服务用于类型转换。
2. 配置属性源，将启动参数作为属性源的一部分。
3. 配置 Profile，默认为空实现。

之后 prepareEnvironment 方法发布 environmentPrepared 事件回调：

```java
// prepareEnvironment 方法
listeners.environmentPrepared(bootstrapContext, environment);
```

随后就调用 bindToSpringApplication 方法将环境对象和 SpringApplication 对象进行绑定：

```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {  
   try {  
      Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));  
   }  
   catch (Exception ex) {  
      throw new IllegalStateException("Cannot bind to SpringApplication", ex);  
   }  
}
```

绑定完成后检查当前环境对象是否为自定义对象，如果不是则进行转换，将创建一个新的环境对象并拷贝原环境对象的属性并返回：

```java
if (!this.isCustomEnvironment) {  
   environment = convertEnvironment(environment);  
}
```

## 打印 Banner

完成环境对象的创建后，则开始调用 printBanner 方法在控制台打印 banner。banner 有两种形式：图片和文本。

图片 banner 从以下途径获取：

+ 尝试从 `spring.banner.image.location` 属性获取路径。
+ 尝试从类路径下获取名称为 banner，后缀为 gif、png 或者 jpg 的图片。

文本 banner 从以下途径获取：

+ 尝试从 `spring.banner.location` 或者 `banner.txt` 属性获取文本文件路径

当以上两种方式可以获取到 banner 对象时，则直接使用。否则，使用 Spring Boot 默认的 banner 对象。得到 Banner 对象后，回调对象的 printBanner 打印 banner。

## 创建上下文

在 run 方法中通过调用 createApplicationContext 方法进行上下文的创建，此时也是通过应用程序的类型来创建出对应的上下文对象。以 Web 应用程序为例，此时返回的上下文对象类型为 AnnotationConfigServletWebServerApplicationContex：

```java
context = createApplicationContext();
```

在获取到新的上下文对象后，也是需要对其进行前置准备工作的：

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,  
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,  
      ApplicationArguments applicationArguments, Banner printedBanner) {  
   context.setEnvironment(environment);  // 1
   postProcessApplicationContext(context);  // 2
   applyInitializers(context);  // 3
   listeners.contextPrepared(context);  // 4
   bootstrapContext.close(context);  
   if (this.logStartupInfo) {  
      logStartupInfo(context.getParent() == null);  
      logStartupProfileInfo(context);  
   }  
   // Add boot specific singleton beans  
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();  
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);  // 5
   if (printedBanner != null) {  
      beanFactory.registerSingleton("springBootBanner", printedBanner);  // 6
   }  
   if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {  
      ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);  
      if (beanFactory instanceof DefaultListableBeanFactory) {  
         ((DefaultListableBeanFactory) beanFactory)  
               .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);  
      }  
   }  
   if (this.lazyInitialization) {  
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());  
   }  
   // Load the sources  
   Set<Object> sources = getAllSources(); // 7
   Assert.notEmpty(sources, "Sources must not be empty");  
   load(context, sources.toArray(new Object[0]));  // 8
   listeners.contextLoaded(context);  
}
```

1. 首先将环境对象设置到上下文中。
2. 后置处理主要将环境对象中的转换服务设置到 BeanFactory 中去。
3. 回调所有 ApplicationContextInitializer 对象的 initialize 方法。
4. 监听器发布 contextPrepared 事件。
5. 向 BeanFactory 中注册名称为 springApplicationArguments 的单例 Bean，该 Bean 由启动参数包装而来。
6. 如果打印 banner 存在时，则向 BeanFactory 中注册名称为 springBootBanner 的单例 Bean，该 Bean 由启动参数封装而来。
7. 从 SpringApplication 中加载源，默认将主启动类作为源。
8. 将得到的源转换为 Bean 定义信息，然后注册到 BeanFactory 中。
9. 监听器发布 contextLoaded 事件。

## 刷新上下文

准备完成上下文之后就是进入刷新上下文的阶段。调用 refreshContext 方法进行刷新：

```java
private void refreshContext(ConfigurableApplicationContext context) {  
   if (this.registerShutdownHook) {  
      shutdownHook.registerApplicationContext(context);  
   }  
   refresh(context);  
}
```

可以看到在刷新上下文之前，它还在关闭钩子中注册 ApplicationContext 的回调。

完成上下文刷新后，此时整个 Spring Boot 程序算是启动完成了。随后监听器就发布了 started 事件：

```java
listeners.started(context, timeTakenToStartup);
```

## 完成后回调

上下文刷新完成后，调用 callRunners 方法进行最后的回调：

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {  
   List<Object> runners = new ArrayList<>();  
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());  // 1
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());  // 2
   AnnotationAwareOrderComparator.sort(runners);  
   for (Object runner : new LinkedHashSet<>(runners)) {  
      if (runner instanceof ApplicationRunner) {  
         callRunner((ApplicationRunner) runner, args);  
      }  
      if (runner instanceof CommandLineRunner) {  
         callRunner((CommandLineRunner) runner, args);  
      }  
   }  
}
```

1. 获取所有的 ApplicationRunner 类型的 Bean 进行回调。
2. 获取所有的 CommandLineRunner 类型的 Bean 进行回调。