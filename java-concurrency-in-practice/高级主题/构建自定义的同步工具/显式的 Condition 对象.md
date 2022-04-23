# 显式的 Condition 对象
#Java #Multithreading 

正如 Lock 是一种广义的内置锁，Condition 也是一种广义的内置条件队列。内置条件队列存在的缺陷：

+ 每个内置锁都只能有一个相关联的内置条件队列。
+ 多个线程在同一内置条件队列下等待不同的条件谓词。
+ 常见的加锁模式会公开条件队列对象。

因此，如果想编写一个带有多个条件谓词的并发对象，或者想获得除了条件队列可见效之外更多的控制权，就可以使用显式的 Lock 和 Condition 而不是内置锁和条件队列，这是一种更加灵活的选择。

一个 Condition 对象和一个 Lock 对象相关联，而一个 Lock 对象可以关联多个 Condition 对象。Condition 对象的创建通过关联的 Lock 对象的 newCondition 方法。Condition 比内置条件队列提供了更丰富的功能：在每个锁上可存在多个等待、条件等待可以是可中断的或不可中断的、基于时限的等待，以及公平的或非公平的队列操作。

Condition 对象继承了相关的 Lock 对象的公平性，对于公平的锁，线程会依照 FIFO 顺序从 Condition.await 方法中释放。Condition中使用 await、signal 和 signalAll 方法来代替内置条件队列 wait、notify 和 notifyAll 方法。但由于对象要继承于 Object，因此一定要确保使用正确的版本。

例如，使用显式条件变量的有界缓存：

```java
public class ConditionBoundedBuffer<T> {

    protected final Lock lock = new ReentrantLock();

    // 条件谓词：notFull
    private final Condition notFull = lock.newCondition();

    // 条件谓词：notEmpty
    private final Condition notEmpty = lock.newCondition();

    private static final int BUFFER_SIZE = 100;

    private final T[] items = (T[]) new Object[BUFFER_SIZE];

    private int tail, head, count;

    public void put(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[tail] = t;
            if (++tail == items.length) {
                tail = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            T t = items[head];
            items[head] = null;
            if (++head == items.length) {
                head = 0;
            }
            --count;
            notFull.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }
}
```

由于使用不同的 Condition 对象可将不同的条件谓词分开并放入两个等待线程集中，Condition 使其更容易满足单次通知的需求。由于 Condition 对象中只包含一个条件谓词时，使用 signal 比 signalAll 方法更高效，它能极大地减少在每次缓存操作中发生的上下文切换与锁请求的次数。

与内置锁和条件队列一样，当使用显式的 Lock 和 Condition 时，也必须满足锁、条件谓词和条件变量之间的三元关系，且检查条件谓词以及调用 await 和 signal 方法时，必须持有 Lock 锁。


