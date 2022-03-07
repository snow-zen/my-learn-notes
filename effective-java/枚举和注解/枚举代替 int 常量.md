# 枚举代替 int 常量
#Java 

枚举类型是一组固定常量组成合法值的类型。在 Java 引入枚举类型之前，通常是一组 int 常量来表示枚举类型，其中每一个 int 常量表示枚举类型的一个成员：

```java
public static final int APPLE_FUJI = 0;
public static final int APPLE_PIPPIN = 1;

public static final int ORANGE_NAVEL = 0;
public static final int ORANGE_TEMPLE = 1;
```

这种形式被称为 **int 枚举模式**，它存在许多不足点：
+ 不具备类型安全，也没有描述性可言。例如，将 APPLE_FUJI 与 ORANGE_NAVEL 混用编译器也不会报错。
+ 不同 int 常量需要加前缀防止命名冲突。
+ 如果 int 常量值发生变化则引用它的客户端也要跟着一起变化，重新编译。
+ 很难将 int 枚举常量转换为可打印的字符串，且打印出来也没有用处。
+ 遍历 int 枚举常量以及获取枚举常量数组大小时，几乎没有可靠的方式。

这种形式还有一种变体，使用 String 常量，被称为 String 枚举模式，但它同样也不是我们期望的。虽然它提供了可打印的字符串，但是会导致初级用户直接把字符串常量硬编码到客户端代码中，一旦这种硬编码包含书写错误，则在编译时也不会被检测到，但是会在运行时报错。

Java 枚举类型的基本想法非常简单：这些类通过公有静态 final 属性为每个枚举常量导出一个实例。枚举类型没有可访问的构造方法，所以它是真正的 final 类。另外，客户端也不能创建枚举类型的实例，也不能对它进行扩展，因此不存在实例，而只存在声明过的枚举常量。

例如：

```java
public enum Apple { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

相比之前的枚举模式，枚举类带来以下好处：
+ 每个枚举常量都是公有静态 final 属性，且没有可访问的构造方法，是真正的 final 类。
+ 保证了编译时的类型安全，任何枚举类型的非空对象都一定是定义的枚举常量之一。
+ 每个枚举类型都有自己的命名空间，包含多个同名常量的多个枚举类型可以共同存在。
+ 允许添加任意方法和熟悉，并实现接口。
+ 如果枚举常量创建后被删除，则引用的客户端会在这行抛出异常。

> 如果一个枚举具有普遍适应性，则它就应该成为一个顶层类。通过将适应性强的类声明为顶层类，从而增强 API 之间的一致性。

当需要在枚举常量中携带数据时，得先声明实例，并编写一个带有数据并将数据保存在属性中的构造方法：

```java
public enum Planet {
	MERCURY(3.302e+23, 2.439e6),
	VENUS(4.869e+24, 6.052e6);

	private final double mass;
	private final double radius;

	Planet(double mass, double radius) {
		this.mass = mass;
		this.radius = radius;
	}
}
```

枚举类还可以与不同代码行为关联起来：在枚举类中声明一个抽象方法，并在特定与常量的类主体中实现具体方法，这种方法被称为**特定于常量的方法**实现：

```java
public enum Operation {
	PLUS("+") {
		public double apply(double x, double y) {
			return x + y;
		}
	}

	MINUS("-") {
		public double apply(double x, double y) {
			return x - y;
		}
	}

	TIMES("*") {
		public double apply(double x, double y) {
			return x * y;
		}
	}

	DIVIDE("/") {
		public double apply(double x, double y) {
			return x / y;
		}
	}

	private final String symbol;

	public abstract double apply(double x, double y);

	Operation(String symbol) {
		this.symbol = symbol;
	}
}
```

这种模式可根据所选枚举常量的不同进行各种不同的行为，这种模式被称为“策略枚举”。但如果是外部枚举不受控制的情况下，可以使用 switch 语句给外部枚举类型增加特定于常量的行为：

```java
public static Operation inverse(Operation op) {
	switch(op) {
		case PLUS: return Operation.MINUS;
		case MINUS: return Operation.PLUS;
		case TIMES: return Operation.DIVIDE;
		case DIVIDE: return Operation.TIMES;
		default: throw new AssertionError("Unknow op:" + op);
	}
}
```

一般来说，枚举通常在性能上与 int 常量相当，虽然相比之下它在装载和初始化枚举时会额外需要空间和时间的成本，但在实践中几乎注意不到这个问题。因此，每当需要一组固定常量，并且在编译时就知道其成员的时候，就应该使用枚举。

另外，枚举中的常量集并不一定要始终保持不变，专门设计枚举特性是考虑到枚举类型的二进制兼容演变的。