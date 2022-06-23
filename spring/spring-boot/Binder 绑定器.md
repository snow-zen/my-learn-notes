# Binder 绑定器
#Java #Spring 

Binder 也是从 Spring Boot 2.0 中新引入的接口，代替原有的 ConfigurableEnvironment 所提供的属性绑定功能。Binder 依赖于同在 Spring Boot 2.0 中新引入的接口 ConfigurationPropertySource，通过属性名称获取到对应的 ConfigurationPropertySource 进行属性值重新绑定或创建。

## 创建 Binder 对象

Binder 提供两种方式创建对象：构造方法和静态工厂方法。

构造方法需要传入的参数较多，但可以更加灵活的控制：

```java
public Binder(Iterable<ConfigurationPropertySource> sources, PlaceholdersResolver placeholdersResolver,  
      List<ConversionService> conversionServices, Consumer<PropertyEditorRegistry> propertyEditorInitializer,  
      BindHandler defaultBindHandler, BindConstructorProvider constructorProvider) {
	// ...
}
```

而静态方法更像是对 Environment 对象提供的一种便捷创建的方式：

```java
public static Binder get(Environment environment) {  
   return get(environment, null);  
}
```

## Bindable

创建完成 Binder 对象之后，在对属性进行绑定时，对应的属性值则需要通过 Bindable 对象进行包装。Bindable 可包装现有的对象、类型信息或者复杂的类型（Collection、Map）等，例如：

```java
Bindable.ofInstance(existingBean);
Bindable.of(Integer.class);
Bindable.listOf(Person.class);
Bindable.of(resovableType);
```

需要注意的是，Bindable 类是一个不可变类，其所有的属性都使用 final 修饰，因此每次调用方法修改状态实际上是创建新的 Bindable 对象。

## BindResult

在使用 Binder 对象对通过 Bindable 包装的示例进行绑定时，方法会返回一个 BindResult 对象。BindResult 类的使用方法与 JDK 8 中的 Optional 对象非常类似，它表示可能已绑定或未绑定的内容。

例如，为 person.data-of-birth 属性绑定值：

```java
BindResult bound = binder.bind("person.date-of-birth",
	Bindable.of(LocalDate.class));

// Return LocalDate or throws if not bound
bound.get();

// Return a formatted date or "No DOB"
bound.map(dateFormatter::format).orElse("No DOB");

// Return LocalDate or throws a custom exception
bound.orElseThrow(NoDateOfBirthException::new);
```

## ConversionService

当调用构造方法创建 Binder 对象时，可传入 ConversionService 类型列表对象。大多数情况下属性名所获取到的 ConfigurationPropertySource 对象内的基础值为 String 类型，此时可通过 ConversionService 类型对象进行转换。

## BindHandler

BindHandler 类型定义多个回调方法，如果需要在绑定时实现额外的逻辑，则可实现该接口并在创建时传入对应的对象即可使用。