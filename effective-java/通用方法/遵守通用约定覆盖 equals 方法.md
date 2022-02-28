# 遵守通用约定覆盖 equals 方法
#Java 

覆盖 equals 方法操作虽然简单，但是错误的覆盖方式所造成的后果非常严重。

避免问题产生最简单方案就是不覆盖 equals 方法。这种情况下，类实例都是与它自身相等，以下几种情况正好符合要求：
+ 类的每个实例都是唯一的。例如：Thread。
+ 类没有必要提供“值相等”的功能。
+ 父类已经覆盖了 equals 方法且对于子类也是合适的。
+ 类本身是私有或包级私有的，并且可以确认它的 equals 方法不会被调用。

## 覆盖 equals 方法时机

当类自身具有“逻辑相等”概念，而且父类没有覆盖 equals 方法。类通常是一个值类，程序员通常希望了解它们在逻辑上是否相等，而不是想了解它们是否指向同一实例。

另一种特殊的值类是不需要覆盖 equals 方法，它们每个值至多只存在一个实例，例如：枚举类。这种类逻辑相等和实例相等是同一概念，因此无需覆盖 equals 方法。

## 覆盖 equals 方法通用约定

覆盖 equals 方法时需要遵守 Object 类中定义的等价关系，其规则如下：
+ **自反性**：对于任何非 null 的引用值 x，x.equals(x) 必须返回 true。
+ **对称性**：对于任何非 null 的引用值 x 和 y，当且仅当 x.equals(y) 返回 true 时，y.equals(x) 也必须返回 true。
+ **传递性**：对于任何非 null 的引用值 x、y 和 z，如果 x.equals(y) 返回 true，y.equals(z) 返回 true 时，那么 x.equals(z) 也必须返回 true。
+ **一致性**：对于任何非 null 的引用值 x 和 y，只要 equals 操作所用到的实例没有发生修改，则多次调用 x.equals(y) 就会一致返回 true 或 false。
+ 对于任何非 null 的引用值 x，x.equals(null) 必须返回 false。

> 相等的元素之间是存在同一等价关系内的。

## 等价关系

不严格的说，等价关系是一个操作符。它将一组元素划分到其元素与另一个元素等价的分组中，这些分组被称作等价类。从客户端的角度看，在使用 equals 方法时，每个等价类中的所有元素都是必须可交换的。

### 自反性

实例必须等于自身，这一条规则很难在无意识的情况下违背。

> ⚠️ 注意：如果违背该规则，在实例加入容器后，再调用容器的 contains 方法会返回 false 表示容器中不存在该实例。

### 对称性

任何两个实例对于“它们是否相等”这一问题的结果必须保持一致。

例如，自定义类支持与 String 类型比较：

```java
public class CaseInsensitiveString {
    
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = s;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj instanceof CaseInsensitiveString) {
            return s.equalsIgnoreCase(((CaseInsensitiveString) obj).s);
        }
        if (obj instanceof String) {
            return s.equalsIgnoreCase((String) obj);
        }
        return false;
    }
}
```

例子中虽然 equals 方法的意图很好，试图与普通的 String 类型互相比较值。在使用 CaseInsensitiveString 实例与 String 实例进行互相比较时，我们却得到不一致的结果：

```java
CaseInsensitiveString cis = new CaseInsensitiveString("test");
String s = "test";

System.out.println(cis.equals(s)); // true
System.out.println(s.equals(cis)); // false
```

这里的问题在于使用 `s.equals(cis)` 时，由于 s 判断 cis 并不是 String 类型所以才返回的 false，这个结果明显违反了对称性规则。

> ⚠️ 注意：一旦违反了对称性规则，当其他实例遇到你的实例时，你完全不知道这些实例的行为会怎么样。

### 传递性

规则要求第一个实例等于第二个实例，第二个实例等于第三个实例时，第一个实例一定等于第三个实例。

例如，一个坐标类：

```java
public class Point {

    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Point)) {
            return false;
        }
        Point p = (Point) obj;
        return p.x == x && p.y == y;
    }
}
```

当 Point 类被继承并增加额外属性，且子类没有重新覆盖 equals 方法时，则在使用 equals 方法比较时会将子类中额外添加的信息忽略。虽然这样做不会违反 equals 约定，但明显是无法接受。

例如，子类实现添加颜色属性：

```java
public class ColorPoint extends Point {

    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof ColorPoint)) {
            return false;
        }
        return super.equals(obj) && ((ColorPoint) obj).color == color;
    }
}
```

ColorPoint 子类中覆盖的 equals 方法只有当参数类型为 ColorPoint ，且具有相同的位置和颜色时，才能返回 true。这个方法的问题在于它违反了对称性规则，在比较实例互换时，可能会得到不同的结果：

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);

System.out.println(p.equals(cp)); // true
System.out.println(cp.equals(p)); // false
```

而如果我们在使用 equals 方法进行类型混合比较时，忽略颜色信息：

```java
@Override
public boolean equals(Object obj) {
    if (!(obj instanceof Point)) {
        return false;
    }
    if (!(obj instanceof ColorPoint)) {
        return obj.equals(this);
    }
    return super.equals(obj) && ((ColorPoint) obj).color == color;
}
```

这种方式虽然符合对称性规则，但是却牺牲了传递性规则：

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);

System.out.println(p1.equals(p2)); // true
System.out.println(p2.equals(p3)); // true
System.out.println(p1.equals(p3)); // false
```

前两次比较中只比较了位置信息，而第三次比较中额外考虑了颜色信息。并且这种方式还可能存在无限递归的问题：假设 Point 存在两个子类 ColorPoint 和 SmellPoint，它们各自都使用使用这种方式覆盖 equals 方法，那么对于 colorPoint.equals(smellPoint) 调用将会抛出 StackOverflowError 异常。

**如何解决传递性问题？**

事实上，这是面向对象语言中关于等价关系的一个基本问题。我们无法在扩展可实例化类的同时，又保留 equals 约定。或许你听过在 equals 方法中使用 getClass 方法代替 instanceof 关键字判断，可以扩展实例化类和增加新的值组件，同时又保留 equals 约定：

```java
@Override
public boolean equals(Object obj) {
    if (!(obj.getClass() == this.getClass())) {
        return false;
    }
    Point p = (Point) obj;
    return p.x == x && p.y == y;
}
```

但这种方式只有当实例之间属于相同实现类时，才为等同。虽然不会太糟糕，但是结果却是无法接受的：Point 子类的实例仍然是一个 Point 类实例，它仍然需要发挥作用，而采取这种方式它是无法完成任务的。同时这种方式也不符合[[里氏代换原则]]，在一些场景中无法代替父类完成任务。例如，在元素类型为 Point 的集合中调用 contains 方法传入 ColorPoint 始终返回 false。

虽然没有一种令人满意的办法可以做到，但还是有些不错的权宜之计。例如，遵守[[复合优于继承]]，让 ColorPoint 不再继承 Point，而是在 ColorPoint 中加入一个私有 Point 属性，以及一个公有的视图方法：

```java
public class ColorPoint {

    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        this.point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof ColorPoint)) {
            return false;
        }
        ColorPoint cp = (ColorPoint) obj;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```

需要注意的是，上述讨论都是建立父类可以被实例化的情况下可能发生的问题。如果是在一个抽象类中增加新的值组件，是不违反 equals 约定的。因此，只要不是直接创建父类的实例，前面所述的问题都不会发生。

### 一致性

如果两个实例相等时，它们必须始终保持相等，直到它们中一个被修改为之。

当编写一个类时，应该仔细考虑类是否不可变。如果类是可变的，则类实例可以不同的时候与不同的实例相等；如果类是不可变的，就必须保证 equals 方法满足限制：相等的实例永远相等，不相等的实例永远不相等。

同时无论类是否不可变，都不要使 equals 方法依赖于不可靠的资源去做判断。为了避免发生这种问题，equals 方法应该对驻留内存的实例执行确定计算。

### 非空性

参与 equals 方法判断的实例都不能为 null，同时通用约定不允许抛出 NPE 异常。对于非 null 值这点无需过多的测试，因为一般在真正比较前，我们都会进行类型检查：

```java
@Override
public boolean equals(Object obj) {
    if (!(obj instanceof ColorPoint)) {
        return false;
    }
    // ...
}
```

使用 instanceof 判断时，如果传入的实例为 null，则必定返回 false。

## 高质量 equals 诀窍

结合上述的约定，可以得出以下实现高质量 equals 方法的诀窍：
1. 使用 == 操作符在开始处直接检查实例引用。如果为 true，则直接返回结果。
2. 使用 instanceof 操作符检查参数是否为对应类型。
3. 把参数转换为正确的类型。
4. 对于该类中每个关键属性，检查参数实例的属性是否和当前实例的属性是否一致。

需要注意的点是：
1. 对于非 float 和 double 的基本类型，可以直接使用 == 操作符比较。
2. 对于 float 和 double 类型可以调用对应包装类的 compare 方法比较。
3. 对于引用类型实例可以递归调用 equals 方法进行比较。

额外告诫：
1. [[覆盖 equals 方法时始终要覆盖 hashCode 方法]]。
2. 不要企图让 equals 方法过于智能。
3. 不要将 equals 声明中的 Object 实例替换为其他类型。

大多数情况下，覆盖 equals 和 hashCode 方法的工作都是十分琐碎的，得到的代码也是十分琐碎的。因此，可以使用 Google 的 AutoValue 框架，它会自动代替你生成方法，或者 IDE 也会有对应的功能。但是生成的方法代码会更加的冗长，可读性也更差，同时它也无法自动追踪类中的变化，因此需要进行测试。

总之，不要轻易覆盖 equals 方法，除非迫不得已。