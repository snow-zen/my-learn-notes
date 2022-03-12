# 返回零长度的数组或集合，而不是 null
#Java 

对于一个返回集合类型的方法来说，在实际返回时返回 null 是一种不好的做法。因为每次客户端接收返回值时，都需要去判断是否为 null 的情况，如果没有进行判断则很容易发生错误。

> null 并不是零长度的容器，它会使客户端代码变得更加复杂。

有人可能会认为反复创建零长度的容器可能会影响性能，但我们也可以通过重复返回同一个不可变的零长度容器，来避免对象的分配重复执行。例如，Collections.emptyList、Collections.emptySet。

数组也是和集合容器一样的，永远都不要返回 null，而是返回零长度的数组：

```java
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];

public Cheese[] getCheeses() {
	return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

总之，永远不要返回 null，而不返回一个零长度的数组或者集合。如果返回 null，那样会使 API 更难以使用，也更容易出错，而且没有任何性能优势。