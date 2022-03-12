# 为所有导出的 API 元素编写文档注释
#Java 

Java 编程环境提供了一种被称为 Javadoc 的实用工具，从而使这项任务变得容易。为了正确地编写 API 文档，必须在每个被导出的类、接口、构造方法、方法和属性声明之前增加了一个文档注释。

方法的文档注释应该简洁地描述它和客户端之间的约束。除了专门为继承而设计的类方法之外，这个约定应该说明这个方法做了什么，而不是说明它是如何完成这项工作。

方法的文档注释需要简洁地描述以下几点：

1. 它和客户端之间的约束。
2. 方法做了什么工作，而不是如何做的。
3. 列举这个方法的前提条件和后置条件。
4. 方法的副作用。

在 Java 8 中增加了 `@implSpec` 标签，主要用于设计继承类时，必须在文档中注释描述该类与子类之间的约定：

```java
/**
 * Returns true if this collection is empty
 * 
 * @implSpec This implementation returns {@code this.size() == 0}
 * @return true if this collection is empty
 */
public boolean isEmpty() { ... }
```

文档注释在源代码和产生的文档中都应该是易于阅读的，如果无法让两者都易读，则产生的文档的可读性要优于源代码的可读性。

每个文档注释的第一句话是对注释元素的概要描述，概要描述必须简单、准确的描述出目标元素的功能。为了避免混淆，同一个类或者接口中的两个成员或者构造方法，都不应该有相同的概要描述，尤其是重载方法。

当为泛型编写文档注释时，要确保在文档中说明所有的类型参数：

```java
/**
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 */
public interface Map<K, V> { ... }
```

当为枚举类型编写文档时，要确保在文档中说明常量，以及任何公有的方法：

```java
/**
 * An instrument section of a symphony orchestra.
 */
public enum OrchestraSection { 
    /** Woodwinds, such as flute, clarinet, and oboe. */
	WOODWIND;
}
```

当为注解类编写文档时，也要确保在文档中说明所有成员，包括类型本身。

包级私有的文档注释应该放在一个 package-info.java 的文件中，如果使用模块系统，应该将模块级的注释放在 module-info.java 文件中。

类或者静态方法是否线程安全，应该在文档中对它的线程安全级别进行说明。API 的两个特性在文档中经常被忽视，即线程安全性和可序列化性。

Javadoc 还具有继承功能，可通过 `{@inheritDoc}` 从父类中继承文档注释的部分内容，以减少维护多个几乎相同的文档注释的负担。

> 编写文档注解最权威的指导仍然是《[How to Write Doc Comments](https://www.oracle.com/technical-resources/articles/java/javadoc-tool.html)》。