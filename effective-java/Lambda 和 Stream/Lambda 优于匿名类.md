# Lambda 优于匿名类
#Java 

用带有单个抽象方法的接口作为函数类型，它们的实例称之为**函数实例**，表示函数或者要采取的动作。Java 中的匿名类满足了传统的面向对象的设计模式对函数对象的需求，最著名的有[[策略模式]]。Java 中的 Comparator 接口代表一种排序的抽象策略，而具体的匿名类就是具体策略。

匿名类的不足点在于创建语法的烦琐使得在 Java 中进行函数式编程前景十分暗淡。但在 Java 8 中对“只带单个抽象方法的接口”进行了特殊对待，这些接口被称作**函数接口**。Java 允许使用 Lambda 表达式创建这些函数接口的实例，Lambda 表达式类似于匿名函数，但语法上则更加简洁。

例如，使用 Lambda 表达式定义排序规则进行排序：

```java
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

在 Lambda 表达式中无需指定参数、返回值类型，编译器可以通过一个称作类型推导的过程，根据上下文推断出这些类型。虽然某些情况下，编译器无法确定类型则必须显式指定，但这种情况很少见。

> 由于编译器是通过从泛型中获得的信息，用于执行类型推断的。因此如果你没有提供这些信息，编译器就无法进行类型推断。

另一方面，Lambda 表达式在实现枚举[[枚举代替 int 常量|特定于常量的方法]]也更加容易，代码也更加的简单和清晰：

```java
public enum Operation {
	PLUS("+", (x, y) -> x + y),
	MINUS("-", (x, y) -> x - y),
	TIMES("*", (x, y) -> x * y),
	DIVIDE("/", (x, y) -> x / y);

	private final String symbol;
	private final DoubleBinaryOperator op;

	Operator(String symbol, DoubleBinaryOperator op) {
		this.symbol = symbol;
		this.op = op;
	}

	public double apply(double x, double y) {
		return op.applyAsDouble(x, y);
	}
}
```

虽然 Lambda 表达式使得代码的编写更加的简单，但是它也是存在一些缺点的：
1. 与匿名类相似，它没有名称和文档。如果一个计算本身不是自描述的，或者超出了几行，那就不要把它放在一个 Lambda 中。一行合理，三行极限，违背原则则会对程序的可读性造成严重的危害，要么简化代码，要么重构。
2. 如果想创建带多个抽象方法的接口实例，则还是需要匿名类完成而不是 Lambda 表达式。
3. Lambda 表达式无法通过 this 关键字获得自身实例的引用，而匿名类可以。

最后需要额外注意的一点是，在使用 Lambda 与匿名类作为属性时，你无法通过序列化和反序列化的方式来进行共享。因此，尽可能不要序列化一个 Lambda 或者匿名类。