# AdvisedSupportListener 监听器
#Java #Spring 

注册在 ProxyCreatorSupport 对象上的监听器，该接口提供两个回调方法：

1. activated 方法：在 ProxyCreatorSupport 首次开始创建 AOP 代理时回调。
2. adviceChanged 方法：在 AOP 代理发生改变时被回调。

实际的接口定义也非常简单：

```java
public interface AdvisedSupportListener {  
  
   void activated(AdvisedSupport advised);  
  
   void adviceChanged(AdvisedSupport advised);  
  
}
```

## AdvisedSupport 类型参数

传入的 AdvisedSupport 类型参数实际上 ProxyCreatorSupport 类型对象本身，因为 ProxyCreatorSupport 继承于 AdvisedSupport 类型。

## 使用方法

以 Spring 注解使用场景为例，根据[[Spring AOP 工作原理和流程]]可以知道所有的代理对象都是由 AnnotationAwareAspectJAutoProxyCreator 这个 Bean 后置处理器在初始化对象后完成的。

其中创建代理对象的 createProxy 核心方法中，会创建 ProxyFactory 类型实例。而 ProxyFactory 类型是 ProxyCreatorSupport 类型的子类型，但是在方法的实现中并没有显式调用 addListener 注册监听器。不过 createProxy 方法在中间调用了一个模版方法 customizeProxyFactory 由子类实现对 ProxyFactory 方法的自定义设置：

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,  
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {
	// ...
	customizeProxyFactory(proxyFactory);
	// ...
	return proxyFactory.getProxy(classLoader);
}
```

可是在实际的实现中 customizeProxyFactory 方法并没有被子类重写，因此如果想要自定义则需要手动继承 AnnotationAwareAspectJAutoProxyCreator 类型并自行注册到 Ioc 容器中。

从[[Spring AOP 工作原理和流程]]中可了解到在注册时，会判断 Ioc 容器中是否存在名为 org.springframework.aop.config.internalAutoProxyCreator 的Bean来判断是否需要注册。因此我们只需要在此之前向其中注册对应名称的 Bean 即可进行自定义操作。



