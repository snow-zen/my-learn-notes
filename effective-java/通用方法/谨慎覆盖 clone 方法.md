# 谨慎覆盖 clone 方法
#Java 

Cloneable 接口通常是对象的一个 mixin 接口，表示实例是允许被克隆的。但是接口没有达到对应的目的，主要缺陷在于缺少 clone 方法，而 Object 类中的 clone 方法是受保护的。在不借助反射的情况下，仅实现了 Cloneable 接口也是不可以调用 clone 方法。

## Cloneable 接口

Cloneable 接口是一个标识接口，它没有任何方法。它主要用于决定 Object 中受保护的 clone 方法的行为：如果类实现了 Cloneable 接口，Object.clone 方法就会返回对象的拷贝，否则抛出 CloneNotSupportedException 异常。

> ⚠️ 注意：这是接口的一种极端非典型用法，也不值得效仿。

通常情况下，实现接口是为了表明类可以有什么特征，而 Closeable 接口改变了父类中受保护方法的行为。

## Object.clone 方法规范

虽然规范没有指出，但实现 Closeable 接口是为了提供一个功能适当、公有的 clone 方法。而为了达到这一目的，类以及父类都必须遵守一个复杂、不可实施的，并且基本上没有文档说明的协议，且这种机制处于语言之外：无需调用构造方法即可创建对象。

Object.clone 方法的通用约定非常弱：
+ 创建和返回对象拷贝，拷贝的精确含义取决于对象的类。一般含义是：对于任何对象 x， `x.clone() != x` 将返回 true，并且 `x.clone().getClass() == x.getClass()` 也将返回 true，但这都不是绝对要求。
+ clone 方法返回对象通过调用 super.clone 方法获取。如果类以及父类遵守这一约定，那么 `x.clone().getClass() == x.getClass()`。
+ clone 方法返回对象不应依赖原对象。为了这种独立性，可能需要在 super.clone 方法返回对象前修改对象属性。

返回对象通过调用 super.clone 方法获得，这种机制类似于自动构造方法调用链，只不过不是强制的。如果类的 clone 方法返回实例不是通过调用 super.clone 方法获得，而是通过调用构造方法获得，而该类的子类在调用时会抛出异常：

```java
class CloneA implements Cloneable {

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return new CloneA();
    }
}

class CloneB extends CloneA {

    @Override
    protected CloneB clone() throws CloneNotSupportedException {
        return super.clone(); // 类型不匹配，抛出异常
    }
}
```

如果例子中的 CloneA 被 final 修饰时，那么上述异常问题则可以被忽略，因为 final 类不会有子类。但不调用 super.clone 方法的 final 类是没有理由实现 Closeable 接口，因此它的实现不依赖 Object 的克隆行为。

## Object.clone 实现方法

调用 super.clone 方法得到原始对象功能的完整克隆，这个对象中的属性等同于被克隆对象的属性。

如果对象属性包含一个基本类型值，或者包含一个指向不可变对象的引用，这种情况下无需任何处理，因为不可变类永远不应该提供 clone 方法，因为只会产生没有意义的克隆。如果对象属性是引用可变对象，需要递归地调用内部可变对象属性的 clone 方法进行克隆。如果对象属性是数组，则可以直接调用数组的 clone 方法进行克隆。

事实上，clone 方法就是另一个构造方法，另外必须确保它不会伤害到原始对象，同时确保正确创建被克隆对象中的约束条件。

> ⚠️ 注意：如果对象属性是 final 修饰，则无法在调用 super.clone 方法初始化后再进行赋值。因此 Cloneable 架构与引用可变对象的 final 属性正常用法是不相兼容的。

如果对象属性内部结构复杂，则可能需要自定义一个深度拷贝的方法在 clone 方法中调用。另外，克隆复杂对象的另一种方式是：先调用 super.clone 方法，然后把结构对象中的所有属性设置为初始状态，之后调用高层方法来重新产生对象的状态。例如 HashMap 初始化后，调用 put 方法。

这种做法虽然简单、合理且优美，但是没有直接调用 clone 方法来的便捷。同时它与整个 Cloneable 架构是对立的，因为它完全抛弃了 Cloneable 架构基础的对象复制机制。

## Object.clone 方法注意事项

1. 实现 clone 方法不应该调用可被子类覆盖的方法，因为克隆对象很可能被子类修改状态。
2. 设计为被继承的类，无论选择哪种实现方式，都不应该实现 Cloneable 接口，这可以使子类具有实现或不实现 Cloneable 接口的自由。
3. 编写线程安全类的 clone 方法时，需要进行同步。

## 代替 Object.clone 方法

很多时刻，我们都是没有必要需要这么复杂的克隆行为，我们可以提供某些其他途径来代替对象拷贝。例如，相比对象拷贝的更好办法是提供一个拷贝构造方法或者拷贝工厂。

例如：

```java
// 拷贝构造方法
public Yum(Yum yum) {
	// ...
}

// 拷贝工厂
public static Yum newInstance(Yum yum) {
	// ...
}
```

这两种做法相比 Cloneable/clone 方法具有更多的优势：
+ 不依赖于某种有风险、语言之外的对象创建机制。
+ 不要求遵守尚未制定好的文档规范。
+ 不会与 final 属性的正常使用发生冲突。
+ 不会抛出不必要的受检查异常。
+ 不需要进行类型转换。

甚至它们可以携带一个参数，参数类型是该类所实现的接口。这种基于接口的拷贝构造器和拷贝工厂允许客户选择拷贝的实现类型，而不是强迫客户接受原始的实现类型。

## 总结

总之，所有的问题都与 Cloneable 接口有关，新的接口不应扩展该接口，同时新的类也不应实现该接口。尽量使用拷贝构造方法和拷贝工厂进行对象的克隆。