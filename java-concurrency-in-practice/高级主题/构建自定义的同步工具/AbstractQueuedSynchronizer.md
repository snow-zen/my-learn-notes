# AbstractQueuedSynchronizer
#Java #Multithreading 

AQS 是一个用于构建锁和同步器的框架，许多同步器都可以通过 AQS 很容易并且高效地构造出来。这个类也是许多同步类的基类，包括：

+ ReentrantLock
+ Semaphore
+ CountDownLatch
+ ReentrantReadWriteLock
+ SynchronousQueue
+ FutureTask

基于 AQS 构建的同步器能带来许多的好处：

1. 极大地减少实现工作。
2. 不必处理多个位置上发生的竞争问题。
3. AQS 充分考虑了可伸缩性。

在基于 AQS 构建的同步器中，最基本的操作包括各种形式的获取操作和释放操作。获取操作是一种依赖状态的操作，并且通常会阻塞。

如果一个类想成为状态依赖的类，它必须拥有一些状态。而 AQS 负责管理同步器类中的状态，它管理了一个整数状态信息，可以通过 getState，setState 以及 compareAndSetState 等 protected 类型方法来进行操作。这个整数可以用于表示任意状态。例如：

+ ReentrantLock 中表示所有者线程已经重复获取该锁的次数。
+ Semaphore 用它来表示剩余的许可数量。
+ FutureTask 用它来表示任务的状态（尚未开始、正在运行、已完成以及已取消）。

同步类中还可保持一些额外的状态变量，例如 ReentrantLock 保存了锁当前所有者信息，这样就能区分某个获取操作是重入的还是竞争的。

根据同步器的不同，获取锁的操作可以是独占的（例如 ReentrantLock），也可以是一个非独占操作（例如 Semaphore）。

AQS 代码模版如下：

```java
boolean acquire() throws InterruptedException {
    while (当前状态不允许获取操作) {
        if (需要阻塞获取请求) {
            如果当前线程不再队列，则将其插入队列
            阻塞当前线程
        } else {
            返回失败
        }
    }
    可能更新同步器的状态
    如果线程位于队列中，则将其移出队列
    返回成功
}

void release() {
    更新同步器的状态
    if (新的状态运行某个被阻塞的线程获取成功) {
        解除队列中一个或多个线程的阻塞状态
    }
}
```

对于支持独占的获取操作的同步器，需要实现一些保护方法 tryAcquire、tryRelease 和 isHeldExclusively 等方法。对于支持共享获取的同步器，则应该实现 tryAcquireShared 和 tryReleaseShared 等方法。

tryAcquireShared 返回一个负值，那么表示获取操作失败，返回零值表示同步器通过独占方式被获取，返回正值则表示同步器通过非独占方式被获取。tryRelease 和 tryReleaseShared 返回 true 时表示释放操作使得所有在获取同步器时被阻塞的线程恢复执行。

例如，使用 AQS 实现的二元闭锁：

```java
public class OneShotLatch {

    private final Sync sync = new Sync();

    public void signal() {
        sync.releaseShared(0);
    }

    public void await() throws InterruptedException {
        sync.acquireInterruptibly(0);
    }

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected int tryAcquireShared(int arg) {
            // 如果闭锁是开的（state == 1），那么这个操作成功，否则将失败。
            return (getState() == 1) ? 1 : -1;
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            // 现在打开闭锁
            setState(1);
            // 现在其他的线程可以获取该闭锁
            return true;
        }
    }
}
```

OneShotLatch 类也可直接扩展 AQS 来实现，而不是将一些功能委托给 AQS，但通过继承的方式来扩展功能并不合理。因为继承会破坏接口的简洁性，继承后 AQS 复杂的公共方法会很容易误用它们。

> java.util.concurrent 中的所有同步器都没有直接扩展 AQS，而是都将它们的相应功能委托给私有的 AQS 子类来实现。