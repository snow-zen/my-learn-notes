# 优先选择 Stream 中无副作用函数
#Java 

Stream 并不只是一个 API，它是一种基于函数编程的模型。为了获得 Stream 带来描述性和速度，有时还有并行性，必须采用泛型以及 API。

Stream 泛型最重要的部分是把计算构造成一系列变型，每一级结果尽可能靠近上一级结果的纯函数。纯函数是指其结果只取决于输入的函数：它不依赖任何可变的状态，也不更新任何状态。为了做到这一点，传入 Stream 操作的任何函数对象，无论是中间操作还是终止操作，都应该是无副作用的。

Stream 中 forEach 操作是终止操作中最没有威力的，也是对 Stream 最不友好的。forEach 操作是显式迭代无法并行，因此应该只用于报告 Stream 计算的结果，而不是执行计算，又或者把 Stream 计算结果添加到其他集合中。

在将 Stream 的元素集中到一个真正的 Collection 里去时比较简单。有三个这样的收集器：toList、toSet 和 toCollection。同时，通过静态导入 Collections 中所有成员都是明智的，因为这样也可以提升 Stream pipeline 的可读性。

例如，获取单词列表中出现频率前十的条目：

```java
List<String> topTen = freq.keySet().stream()
	.sorted(comparing(freq::get).reversed())
	.limit(10)
	.collect(toList());
```

总之，为了正确使用 Stream 必须了解收集器，最重要的收集器工厂是 toList、toSet、toMap、groupingBy 和 joining。