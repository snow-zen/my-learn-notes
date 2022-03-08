# EnumSet 代替位域
#Java 

如果一个枚举类型的元素主要使用在集合中，一般会使用 int 枚举模式：

```java
public class Text {
	public static final int STYLE_BOLD = 1 << 0;
	public static final int STYLE_ITALIC = 1 << 1;
	public static final int STYLE_UNDERLINE = 1 << 2;
	public static final int STYLE_STRIKETHROUGH = 1 << 3;

	public void applyStyles(int styles) {
		// ...
	}
}
```

这种表示法后续可以使用 OR 位运算符将几个常量合并到一个集合中，称作位域：

```java
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

位域表示法通过允许位操作，有效地执行像联合和交集这样的集合操作。但位域具有 int 枚举常量的所有缺点，甚至更多缺点：
1. 位域以数字形式打印时，翻译位域比翻译简单的 int 枚举常量要困难得多。
2. 遍历位域所表示的所有元素也没有很容易的方法。
3. 编写 API 时，必须要预测最多需要的位数。

Java 在 `java.until` 包中提供了 EnumSet 类来有效地表示从单个枚举类型中提取的多个值的多个集合。这个类实现了 Set 接口，提供了丰富的功能、类型安全性，以及可以从任何其他 Set 实现中得到的互用性。

例如，使用枚举代替位域后的示例：

```java
public class Text {
	public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }

	public void applyStyles(Set<Style> styles) {
		// ...
	}
}
```

其对应的客户端实现如下：
```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

appleStyle 方法使用 `Set<Style>` 方法作为参数，这是考虑到可能会有特殊的客户端需要传递一些其他的 Set 实现。因此，枚举类型可以在集合中进行使用，所以没有理由用位域来表示它。