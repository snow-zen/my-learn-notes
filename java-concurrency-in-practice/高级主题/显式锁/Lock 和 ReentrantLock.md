# Lock 和 ReentrantLock
#Java #Multithreading 

与内置锁不同的是，Lock 提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作，所有加锁和解锁的方式都是显式的。在Lock的实现中必须提供和内置锁相同的内存可见性语义，但在加锁语义、调度算法、顺序保证以及性能特性等方面可以有所不同。

ReentrantLock 实现了 Lock 接口，并提供了与 synchronized 相同的互斥性和内存可见性。在获取 ReentrantLock 时，有着与进入同步代码块相同的内存语义。除此之外，ReentrantLock 还提供了可重入的加锁语义。ReentrantLock 与 synchronized 相比，它还为处理锁的不可用性问题提供了更高的灵活性。

例如，使用 ReentrantLock 来保护对象状态：

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // 执行操作
} finally {
    lock.unlock();
}
```

ReentrantLock 不能完全替代 synchronized 的原因：

1. 它更加“危险”，因为当程序的执行控制离开被保护的代码块时，不会自动清除锁。
2. 如果没有使用 finally 来释放 Lock，当问题出现时很难追踪到最初发生错误的位置，因为没有记录应该释放锁的位置和时间。

## 轮询锁和定时锁

可定时的与可轮询的锁获取模式是由 tryLock 方法实现的，与无条件的锁获取模式相比，它具有更完善的错误恢复机制。

> ⚠️ 注意：在内置锁中，死锁是一个很严重的问题，恢复程序唯一的方式是重新启动程序，而防止死锁的唯一方法就是在构造程序时避免出现不一致的锁顺序。

可定时的与可轮询的锁提供了另一种避免死锁发生的选择，因为 tryLock 方法在锁可用时立即获取锁并返回 true，否则立即返回 false。

例如，通过 tryLock 来避免锁顺序死锁：

```java
public class DeadlockAvoidance {

    private static Random rnd = new Random();

    public boolean transferMoney(Account fromAcct,
                                 Account toAcct,
                                 DollarAmount amount,
                                 long timeout,
                                 TimeUnit unit) throws InterruptedException, InsufficientFundsException {
        long fixedDelay = getFixedDelayComponentNanos(timeout, unit);
        long randMod = getRandomDelayModulusNanos(timeout, unit);
        long stopTime = System.nanoTime() + unit.toNanos(timeout);

        while (true) {
            // 按顺序获取锁，只要有一个锁不可用则会释放已经获取的所有锁并重新开始获取
            if (fromAcct.lock.tryLock()) {
                try {
                    if (toAcct.lock.tryLock()) {
                        try {
                            if (fromAcct.getBalance().compareTo(amount) < 0) {
                                throw new InsufficientFundsException();
                            } else {
                                fromAcct.debit(amount);
                                toAcct.credit(amount);
                                return true;
                            }
                        } finally {
                            toAcct.lock.unlock();
                        }
                    }
                } finally {
                    fromAcct.lock.unlock();
                }
            }

            if (System.nanoTime() < stopTime) {
                return false;
            }
            NANOSECONDS.sleep(fixedDelay + rnd.nextLong() % randMod);
        }
    }

    private static final int DELAY_FIXED = 1;
    private static final int DELAY_RANDOM = 2;

    private static long getFixedDelayComponentNanos(long timeout, TimeUnit unit) {
        return DELAY_FIXED;
    }

    private static long getRandomDelayModulusNanos(long timeout, TimeUnit unit) {
        return DELAY_RANDOM;
    }

    static class DollarAmount implements Comparable<DollarAmount> {

        @Override
        public int compareTo(DollarAmount o) {
            return 0;
        }

        public DollarAmount(int dollars) {
        }
    }


    class Account {

        public Lock lock;

        void debit(DollarAmount d) {
        }

        void credit(DollarAmount d) {
        }

        DollarAmount getBalance() {
            return null;
        }
    }

    class InsufficientFundsException extends Exception {
    }
}
```

在带有时间限制的操作中，tryLock 方法可以指定获取锁的限制时间，超过时间后则会取消锁的获取操作，那么就会使程序提前结束。而内置锁，在开始锁的获取后则无法进行取消。

## 可中断的锁获取操作

可中断的锁获取操作同样能在可取消的操作中使用加锁。lockInterruptibly 方法能够在获得锁的同时保持对中断的响应，并且由于它包含在 Lock 中，因此无须创建其他类型的不可中断阻塞机制。

可中断的锁获取操作的标准结构比普通锁获取操作略微复杂一些，因为需要两个 try-catch 块。例如，可中断的锁获取操作：

```java
Lock lock = new ReentrantLock();

try {
    lock.lockInterruptibly();
    try {
        // 执行可取消的操作
    } catch(InterruptedException e) {
       // 捕获执行操作时的中断异常
    }
} catch(InterruptedException e) {
    // 捕获锁获取时的中断异常
} finally {
    lock.unlock();
}
```

## 非块结构的加锁

我们知道可以通过降低锁的粒度可以提高代码的可伸缩性。

锁分段技术在基于散列的容器中实现了不同的散列链，以便使用不同的锁。我们可以通过采用类似的原则来降低链表中锁的粒度，即为每个链表节点使用一个独立的锁，使不同的线程能独立地对链表的不同部分进行操作。

每个节点的锁将保护链接指针以及该节点中存储的数据，因此当遍历或修改链表时，我们必须持有该节点上的这个锁，直到获得了下一个节点的锁，只有这样才能释放前一个节点的锁。

> 这项技术称之为连锁式加锁或者锁耦合。

