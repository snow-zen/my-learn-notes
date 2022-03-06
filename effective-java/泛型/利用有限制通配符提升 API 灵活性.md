# 利用有限制通配符提升 API 灵活性
#Java 

依据[[列表优于数组|泛型是可变]]的特性，即任何两个类型 `Type1` 和 `Type2`，对应的 `List<Type1>` 和 `List<Type2>` 互相即不是对方的子类型，也不是对方的父类型。但有时候我们又需要更高的灵活性。幸运的是，Java 提供了一种特殊的参数化类型，称作有限制的通配符类型。

例如，使用 extends 关键字修改集合的 pushAll 方法：

```java
public void pushAll(Iterable<? extends E> src) {
	for (E e: src) {
		push(e);
	}
}
```

类型参数 `Iterable<? extends E>` 表示“E 的某个子类型的 Iterable 接口”。

又例如，使用 super 关键字修改集合的 popAll 方法：

```java
public void popAll(Collection<? super E> dst) {
	while (!isEmpty()) {
		dst.add(pop());
	}
}
```

类型参数 `Collection<? super E>` 表示“E 的某个父类集合”。

结论很明显，为了获得最大限度的灵活性，要在表示生产者或消费者的输入参数上使用通配符类型。如果某个输入参数即是生产者，又是消费者，那么通配符类型对你就没有什么好处：因为你需要的是严格类型匹配，这是不用任何通配符而得到的。

> 助记符 PECS 表示 producer-extends，consumer-super。

另外，通配符应只限于参数使用，不要用通配符作为返回类型。而且通配符的使用应该是无形的，如果类的客户端必须考虑通配符类型，类的 API 或许就会出错。

有一种较为特殊的表示，即 Comparable 接口的声明：

```java
public static <T extends Comparable<? super T>> T max(List<? extends T> list) {
	// ...
}
```

在 max 方法中最直接运用 PECS 的是参数 list。它作为生产者传入方法，因此将类型从 `List<T>` 改成 `List<? extends T>`。更灵活的是运用到类型参数 T，最初 T 被用来指定用来扩展 `Comparable<T>`，但是 T 的 comparable 消费 T。因此，参数化类型 `Comparable<T>` 被有限制通配符类型 `Comparable<? super T>` 取代。

还有一种特殊的情况是，如果类型参数只在声明中出现了一次，则可以使用无限制通配符取代它：

```java
public static void swap(List<?> list, int i, int j);
```