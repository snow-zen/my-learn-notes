# 谨慎返回 optional
#Java 

在 Java 8 之前，如果想表示方法在特定情况下无返回实例则有两种方法：一是抛出异常，二是返回 null 实例。但这两个方法都不完美，都存在缺陷。

异常应该根据异常条件保留起来。由于创建异常会捕捉整个堆栈，因此抛出异常的开销通常很高。返回 null 实例则需要客户端编写额外的代码去进行判断 null，否则随时有可能发生 NPE 异常。

返回 null 实例虽然没有异常的缺点，但它有自身的不足。如果方法返回 null，客户端就必须包含特殊的代码来处理返回 null 的可能性，除非程序员可以证明不可能返回 null，否则可能会抛出 NPE 异常。[^1]

[^1]: [[谨慎返回 optional]]

Java 8 中可以通过返回一个 `Optional<E>` 实例代替上述的两个方法。Optional 是一个最多只能存放单个实例的不可变容器，内部可能是非 null 的实例引用，或者什么都没有。

> ⚠️ 注意：在使用 Optional 时永远不要通过返回 Optional 的方法返回 null。因为它彻底违背了 optional 的本意。

Optional 本质上与受检异常相类似，因为它们强迫 API 用户面对没有返回值的现实。抛出异常，或者返回 null 都允许用户忽略这种可能性，从而可能带来灾难性的后果。

在获取 Optional 对象内部所需的真实对象时，可通过 orElseGet 方法指定不存在时代替的默认返回对象，或者通过 orElseThrow 方法在不存在时抛出异常。

在使用 Stream 编程时，经常会遇到 `Stream<Optional<T>>`，为了推动进程还需要包含一个非空 optional 中的所有元素。在 Java 8 中可能这样操作：

```java
streamOfOptional
	.filter(Optional::isPresent)
	.map(Optional::get)
```

在 Java 9 中可以结合 flatMap 方式进行代替，同时它会自动处理不包含元素的 optional 对象：

```java
streamOfOptional
	.flatMap(Optional::stream)
```

对于集合、映射、Stream、数组和 Optional 本身时，都不应该被包装在 Optional 对象中。因为它们本身就有存在表达无返回对象的方式，并且客户端还无须对其进行特殊处理。

同时 Optional 类型对象永远不能作为键、值，或者集合或数组中的元素。只有当无返回结果并且当没有返回结果时客户端必须执行特殊的处理，那么就应该声明该方法返回 Optional 类型。

对于基本类型，永远不应该提供它的基本包装类型的 Optional 对象，类库中存在额外的处理类型：OptionalInt、OptionalLong、OptionalDouble等。

对于将 optional 对象作为类的实例属性，只有在描述那些可能缺失的属性时这种做法才会变得有意义。