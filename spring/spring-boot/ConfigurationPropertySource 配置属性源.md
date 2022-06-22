# ConfigurationPropertySource 配置属性源
#Java #Spring 

ConfigurationPropertySource 接口是 Spring Boot 2.0 所引入的，用于实现之前绑定器的部分宽松绑定规则。该接口的方法也非常简单：

```java
ConfigurationProperty getConfigurationProperty(ConfigurationPropertyName name);
```

该接口还有一个子接口 IterableConfigurationPropertySource 用于访问属性源中所有的属性名称：

```java
Stream<ConfigurationPropertyName> stream();
```

早期的 Environment 类也可以通过额外的 ConfigurationPropertySources 类使其能够支持 ConfigurationPropertySource 类的相关操作：

```java
Iterable<ConfigurationPropertySource> sources =
	ConfigurationPropertySources.get(environment);
```

## ConfigurationPropertyName

该类用于表示属性名，不同于之前非常灵活的宽松绑定，可随意使用风格进行属性获取，ConfigurationPropertyName 在代码中获取属性时，必须使用小写的 kebab-case 风格。即使底层属性源使用驼峰命名法、大写命名等规则。

## ConfigurationProperty

该类表示一个配置属性，属性名称对应的 ConfigurationPropertyName 和属性值以及 Origin 属性源信息。

同时该类中的属性也反向持有 ConfigurationPropertySource 对象的引用。

## Origin

Origin 接口也是 Spring Boot 2.0 中引入的一个新接口，可用于查明属性的来源的确切位置。接口有许多的实现，其中最有用的是 TextResourceOrigin，它可以已加载资源的详细信息，包括行号和列号。