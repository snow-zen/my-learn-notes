# 谨慎使用 Stream
#Java 

在 Java 8 中增加 Stream API 用于简化串行或者并行批量操作，该 API 提供了两个抽象概念：
+ Stream 代表有限或者无限数据的元素。
+ Stream pipeline 代表这些元素的多级计算。

Stream 中的元素可以来自于任何位置，常见来源包括集合、数组、文件、正则表达式匹配器、伪随机数生成器以及其他 Stream，同时也可以来源对象引用或者基本类型。

Stream pipeline 中由一个源 Stream、零个或者多个中间操作、一个终止操作组成。pipeline 操作通常是懒加载的：直到调用终止操作时才开始计算，对于完成终止操作不需要的元素则永远不会计算，这使得无限 Stream 成为可能。

> ⚠️ 注意：不带终止操作的 Stream pipeline 将是一个静默的无操作指令，因此千万不能忘记终止操作。

默认情况下获得的 Stream pipeline 是顺序执行的。要使 pipeline 并发执行，只需在该 pipeline 的任何 Stream 上调用 parallel 方法即可，但是通常不建议这么做。

Stream 操作虽然可以使得代码变得简洁，但也随之降低了代码的可读性，因此滥用 Stream 会使得程序代码更难以读懂和维护。在没有显式类型的情况下，请仔细命名 Lambda 表达式参数，这对于 Stream pipeline 的可读性至关重要。如果可以将部分操作抽取作为一个独立的方法，比在迭代代码中使用而言更为重要。

> ⚠️ 注意：最好避免利用 Stream 来处理 char 值，因为 Java 不支持 char 的 Stream 操作，而是当作 int 值处理。

Stream pipeline 利用函数对象来描述重复计算，而迭代版代码则利用代码块来描述重复的计算。下列工作只能通过函数对象来完成：
+ 从代码块中，可以读取和修改范围内任意局部变量。函数对象只能读取 final 变量，并且不能修改任何 local 变量。
+ 从代码块中，可以从外围方法中 return、continue 和 break 外围循环或者抛出异常，但从 Lambda 中则无法做到对应的工作。

反之，在某些工作中使用函数对象会变得易如反掌：
+ 统一转换元素的序列。
+ 过滤元素的序列。
+ 利用单个操作合并元素顺序。
+ 将元素的序列存放到一个集合中，比如根据某些公共属性进行分组。
+ 搜索满足某些条件的元素的序列。

函数对象在使用时也存在一个缺点：将一个值映射到某个其他的值时，原来的值就丢失了。最好的解决办法是，当需要访问较早阶段的值时，将映射颠倒过来。

在实践中可根据实际情况选择对应的实现方式，也可将两者结合在一起使用。具体选择哪种，没有硬性、速成的规则，但是可以参考一些有意义的启发。