# Stream 要优先用 Collection 作为返回类型
#Java 

许多方法都会返回元素的序列。

在 Java 8 之前，这类方法明显的返回类型是集合接口 Collection、Set、List、Iterable 以及数组类型，一般来说很好确认返回具体哪种类型。但在 Java 8 之后添加了 Stream 类型，本质上导致在选择方法返回类型的任务上变得更复杂了。

由于 Stream 接口没有扩展 Iterable 接口，如果方法返回 Stream 接口时则只能通过 iterator 方法获取迭代器无法直接使用 for-each 循环：

```java
for (ProcessHandle ph: (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator) {
	// doing
}
```

这种客户端代码虽然可行，但是实际使用时过于杂乱、不清晰。更好的解决办法是使用适配器方法：

```java
public static <E> Iterable<E> iterableOf(Stream<E> stream) { // 适配器方法
	return stream::iterator;
}

for (ProcessHandle p: iterableOf(ProcessHandle.allProcesses())) { // 改进方法
	// Process the process ... 
}
```

对于想通过 for-each 语句来遍历序列的程序员来说，返回 Stream 并不满意。同时反过来说，对于想利用 Stream pipeline 的程序员，也会被只提供 Iterable 的 API 搞的束手无策。

如果在编写一个返回对象序列的方法时，就知道它只在 Stream pipeline 中使用，当然就可以放心地返回 Stream 了。同样地，如果方法只在迭代中使用时，则应该返回 Iterable。但如果方法在整个公共 API 中使用时，则这需要分别提供这两种返回类型对应的方法，除非有足够的理由相信大多数用户都想要使用相同的机制。

对于公共的、返回序列的方法，Collection 或者适当的子类型通常是最佳的返回类型。因为 Collection 也是 Iterable 的一个子类型。另外，数组也可以通过 Arrays.asList 和 Stream.of 方法提供了简单的迭代和 stream 访问。

> ⚠️ 注意：如果返回的序列很大，但是能被准确表述，可以考虑实现一个专用的集合。千万别在内存中保留巨大的序列，将它作为集合返回即可。