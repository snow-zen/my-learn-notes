# Profile 配置加载流程
#Java #Spring 

Spring Boot 可以编写多环境配置文件，在启动程序时可以通过指定 Profile 加载对应环境的配置文件。本文主要探究在指定对应 Profile 值后 Spring Boot 如何将对应的配置加载到环境中。

## 获取激活的 Profile

Spring Boot 程序中所有的系统属性和自定义属性都由 Environment 环境对象进行管理的，因此根据 Profile 所加载的一系列属性信息最终也会设置到 Environment 环境中去。

在[[Spring Boot 启动流程]]中我们可以知道环境对象的创建和配置是由 `org.springframework.boot.SpringApplication#prepareEnvironment` 方法处理的。通过代码调试我们得知以下信息：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-01.png)

1. Environment 为子类型 ApplicationServletEnvironment 实现。
2. 实现类中存在属性  activeProfiles 用于记录当前激活的 Profile 列表。

通过在类中搜索得到 activeProfiles 属性值的变更只在 setActiveProfiles 方法中出现，因此我们该方法建立断点，并重启程序。程序如期在断点处终止，通过向上查询调用堆栈，我们可以看到：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-02.png)

setActiveProfiles 方法的调用是在 Environment 对象准备完成后，由 SpringApplicationRunListeners 监听器所回调的 environmentPrepared 方法引起的。而 SpringApplicationRunListeners 类是一个委托类，实际处理是由内部一系列 SpringApplicationRunListener 监听器完成的。通过继续查找调用堆栈，我们可以看到：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-03.png)

实际真正的处理是由 EventPublishingRunListener 监听器的 environmentPrepared 方法处理的，而该方法内部则又存在另一层事件监听器模式：

```java
@Override  
public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,  
      ConfigurableEnvironment environment) {  
   this.initialMulticaster.multicastEvent(  
         new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));  
}
```

该 EventPublishingRunListener 监听器主要在 Spring 上下文刷新前，使用内部的事件多播器来支持将事件发布给早期就存在的 ApplicationListener 监听器对象。通过查看堆栈，可以看到：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-04.png)

此处是由 EnvironmentPostProcessorApplicationListener 监听器收到事件并进行处理，而该监听器在收到 ApplicationEnvironmentPreparedEvent 类型事件对象后，调度到内部的 onApplicationEnvironmentPreparedEvent 方法进行处理，并且对应的堆栈信息显示如下：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-05.png)

可以看到调用 getEnvironmentPostProcessors 方法所获得的 EnvironmentPostProcessor 列表，并且已知当前迭代的 EnvironmentPostProcessor 对象的实际类型是 ConfigDataEnvironmentPostProcessor。

这里引入一个额外的问题，getEnvironmentPostProcessors 方法从什么地方获取到的列表？在后续的[[#EnvironmentPostProcessor 来源]]进行解释。

在 ConfigDataEnvironmentPostProcessor 内部的 postProcessEnvironment 方法中，发现 Profile 信息的读取是由 ConfigDataActivationContext 对象处理的：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-06.png)

而 withProfiles 方法内部则则是通过创建 Profiles 对象时，才确认当前激活的 Profile。Profiles 构造器源码如下：

```java
Profiles(Environment environment, Binder binder, Collection<String> additionalProfiles) {  
   this.groups = binder.bind("spring.profiles.group", STRING_STRINGS_MAP).orElseGet(LinkedMultiValueMap::new);  
   this.activeProfiles = expandProfiles(getActivatedProfiles(environment, binder, additionalProfiles));  
   this.defaultProfiles = expandProfiles(getDefaultProfiles(environment, binder));  
}
```

可以看到，当前激活的 Profile 信息是从 getActivatedProfiles 方法中得到的，而 getActivatedProfiles 方法内部则是调用 getProfiles 方法获取 Profile 信息的：

```java
private Collection<String> getProfiles(Environment environment, Binder binder, Type type) {  
   String environmentPropertyValue = environment.getProperty(type.getName());  // 1
   Set<String> environmentPropertyProfiles = (!StringUtils.hasLength(environmentPropertyValue))  
         ? Collections.emptySet()  
         : StringUtils.commaDelimitedListToSet(StringUtils.trimAllWhitespace(environmentPropertyValue));  
   Set<String> environmentProfiles = new LinkedHashSet<>(Arrays.asList(type.get(environment)));  
   BindResult<Set<String>> boundProfiles = binder.bind(type.getName(), STRING_SET);  // 2
   if (hasProgrammaticallySetProfiles(type, environmentPropertyValue, environmentPropertyProfiles,  
         environmentProfiles)) {  
      if (!type.isMergeWithEnvironmentProfiles() || !boundProfiles.isBound()) {  
         return environmentProfiles;  
      }  
      return boundProfiles.map((bound) -> merge(environmentProfiles, bound)).get();  
   }  
   return boundProfiles.orElse(type.getDefaultValue());  
}
```

在位置 1 是直接从环境中获取，在通过启动参数指定 Profile 时生效；位置 2 则从配置文件中读取，在配置文件中直接指定 Profile 时生效。

## 获取对应 Profile 的属性值

获取属性值是和获取激活的 Profile 同时进行的，在调用完 withProfiles 方法后则调用 processWithProfiles 方法获取到对应配置的 PropertySource 对象：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-07.png)

此时对应的配置信息存储在 ConfigDataEnvironmentContributors 对象中，后续则在 applyToEnvironment 方法中的调用 applyContributor 方法将获取到的 PropertySource 对象放入环境中：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-08.png)

applyContributor 具体代码如下：

```java
private void applyContributor(ConfigDataEnvironmentContributors contributors,  
      ConfigDataActivationContext activationContext, MutablePropertySources propertySources) {  
   this.logger.trace("Applying config data environment contributions");  
   for (ConfigDataEnvironmentContributor contributor : contributors) {  
      PropertySource<?> propertySource = contributor.getPropertySource();  
      if (contributor.getKind() == ConfigDataEnvironmentContributor.Kind.BOUND_IMPORT && propertySource != null) {  
         if (!contributor.isActive(activationContext)) {  
            this.logger.trace(  
                  LogMessage.format("Skipping inactive property source '%s'", propertySource.getName()));  
         }  
         else {  
            this.logger  
                  .trace(LogMessage.format("Adding imported property source '%s'", propertySource.getName()));  
            propertySources.addLast(propertySource);  
            this.environmentUpdateListener.onPropertySourceAdded(propertySource, contributor.getLocation(),  
                  contributor.getResource());  
         }  
      }  
   }  
}
```

## EnvironmentPostProcessor 来源

在使用时，是通过 EnvironmentPostProcessorApplicationListener 类的 getEnvironmentPostProcessors 方法获取 EnvironmentPostProcessor 列表的。

getEnvironmentPostProcessors 方法源码如下：

```java
List<EnvironmentPostProcessor> getEnvironmentPostProcessors(ResourceLoader resourceLoader,  
      ConfigurableBootstrapContext bootstrapContext) {  
   ClassLoader classLoader = (resourceLoader != null) ? resourceLoader.getClassLoader() : null;  
   EnvironmentPostProcessorsFactory postProcessorsFactory = this.postProcessorsFactory.apply(classLoader);  
   return postProcessorsFactory.getEnvironmentPostProcessors(this.deferredLogs, bootstrapContext);  
}
```

其中 postProcessorsFactory 属性是一个 Function 函数接口，由对象创建时传入。在此处设置断点，重新运行程序，查看上级调用栈：

![](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/debug-profile-09.png)

fromSpringFactories 方法源码如下：

```java
static EnvironmentPostProcessorsFactory fromSpringFactories(ClassLoader classLoader) {  
   return new ReflectionEnvironmentPostProcessorsFactory(classLoader,  
         SpringFactoriesLoader.loadFactoryNames(EnvironmentPostProcessor.class, classLoader));  
}
```

到这里就可以知道了，通过 SpringFactoriesLoader.loadFactoryNames 方法从类路径的 spring.factories 文件中加载 EnvironmentPostProcessor 对应的实现类。