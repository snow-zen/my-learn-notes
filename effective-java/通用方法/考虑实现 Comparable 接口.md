# 考虑实现 Comparable 接口
#Java 

Comparable 接口中只存在唯一的方法 compareTo。该方法不但允许进行简单的等同性比较，同时支持顺序比较，除此之外还支持泛型功能。当一个类实现 Comparable 接口时，它就可以和依赖该接口的集合实现进行协同工作。

> 事实上，Java 平台库中所有值类以及枚举类都实现了 Comparable 接口。

## compareTo 方法约定

compareTo 方法通用约定与 equals 方法约定相同：实例与指定实例进行比较，当该实例小于、等于或大于指定实例时，分别返回负整数、零或者正整数。如果由于指定对象的类型而无法与该对象进行比较，则抛出 ClassCastException 异常。

接口的实现类必须符合以下规则：
+ 实现类必须确保所有的 x 和 y 都满足 `x.compareTo(y) == -(y.compareTo(x))`。（这也暗示着当且仅当 `y.compareTo(x)` 抛出异常时，`x.compareTo(y)` 也必须抛出异常。）
+ 实现类必须确保比较关系是可传递的：`x.compareTo(y) > 0` 并且 `y.compareTo(z) > 0` 时，`x.compareTo(z) > 0`。
+ 实现类必须确保 `x.compareTo(y) == 0` 暗示所有的 z 都满足 `x.compareTo(z) == y.compareTo(z)`。
+ 最后强烈建议 `(x.compareTo(y) == 0) == (x.equals(y))`，但这并非绝对必要。一般来说，Comparable 实现类在违反该条件时，都应该明确予以说明。推荐使用这种说法：“注意，该类具有内在的排序功能，但是和 equals 方法不一致”。

与 equals 方法约定不同的是，compareTo 方法不能跨越不同类型进行比较。在比较不同类时，compareTo 可以抛出 ClassCastException 异常。

> ⚠️ 注意：违法 compareTo 约定的类也会破坏其他依赖于比较关系的类。依赖于比较关系的类包括有序集合类 TreeSet 和 TreeMap 以及工具类 Collections 和 Arrays，它们内部都含有搜索和排序算法。

与 equals 方法约定相似在于，compareTo 方法也必须遵守相同于 equals 方法约定所施加的限制条件：自反性、对称性和传递性。同时它也有着相同的约束：无法在新的值组件扩展可实例化类时，同时保持 compareTo 约定，除非愿意放弃面向对象的抽象优势。针对[[遵守通用约定覆盖 equals 方法#传递性|equals 的权宜之计]]也同样适用于 compareTo 方法。

## compareTo 方法实现

compareTo 方法中属性是顺序比较，而不是等同性比较。当比较属性是引用类型时，可以通过递归的调用 compareTo 方法实现。如果类没有实现 Comparable 接口或者需要使用一个非标准的排序关系时，就可以使用一个显式的 Comparator 或者自定义比较器来代替。

> 在 Java 7 版本中，所以包装类型都增加了静态的 compare 方法。另外，在 compareTo 方法中使用 < 或 > 关系操作符是非常烦琐的，并且容易出错，因此不建议使用。

在 Comparable 实现类中如果存在多个关键属性，以什么样的顺序来比较属性也是非常关键的，必须从最关键的属性开始进行比较，逐步降低优先级进行比较：

```java
public int compareTo(PhoneNumber pn) {
	int result = Short.compare(areaCode, pn.areaCode);
	if (result == 0) {
		result = Short.compare(prefix, pn.prefix);
		if (result == 0) {
			result = Short.compare(lineNum, pn.lineNum);
		}
	}
	return result;
}
```

在 Java 8 中，Comparator 接口配置了一组**比较器构造方法**，使得比较器的构造工作变得非常流畅。许多程序员都喜欢这种简洁性，虽然它需要付出一定的成本：

```java
private static final Comparator<PhoneNumber> COMPARATOR = 
		comparingInt((PhoneNumber pn) -> pn.areaCode)
				.thenComparingInt(pn -> pn.prefix)
				.thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
		return COMPARATOR.compare(this, pn);
}
```

## compareTo 方法实现注意事项

例如，使用 `-` 操作符对散列值进行比较：

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
	public int compare(Object o1, Object o2) {
		return o1.hashCode() - o2.hashCode();
	}
}
```

千万不要有这种写法，它很容易造成整型溢出，同时也违法 IEEE 754 浮点算法标准，它甚至还没有比其他方法明显变快。

因此，要么使用静态 compare 方法或者比较器构造方法。