# for-each 循环优先于传统的 for 循环
#Java 

例如，使用传统 for 循环或 for-each 循环：

```java
// 迭代器
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    // doing ...
}

// 索引
for (int i = 0; i < a.length; i++) {
    // doing ...
}
```

例子中的两种迭代方式都比 while 循环要更好，但是它们也还是存在问题：

1. 迭代过程中，除元素外还有其他局部变量存在，容易造成混乱。
2. 迭代过程中，元素所在容器类型转移了不必要的注意力，并且为修改类型增加一定难度。

for-each 语句解决了上述的问题，该语句适用于数组、集合以及所有实现 Iterable 接口的实现，简化了一种类型到另一种类型的转化过程：

```java
for (Element el: elements) {
    // doing ...
}
```

但遗憾的是有三种场景无法使用 for-each 循环：
+ 解构过滤，当需要对集合元素进行删除、过滤时。
+ 转换，当需要对集合元素从一种类型转换到另一种类型时。
+ 并行迭代，当需要并行遍历集合或者同时遍历多个集合时。