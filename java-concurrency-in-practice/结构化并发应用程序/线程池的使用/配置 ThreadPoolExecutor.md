# 配置 ThreadPoolExecutor
#Java #Multithreading 

除了 Executors 所提供的现成的线程池实现，我们也可以通过 ThreadPoolExecutor 来进行各种定制。

ThreadPoolExecutor 是一个灵活的、稳定的线程池，允许进行各种定制。当默认的执行策略不能满足需求时，那么可以通过 ThreadPoolExecutor 的构造函数根据自己的需求进行定制。

ThreadPoolExecutor 的通用构造函数如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) { ... }
```

## 线程的创建和销毁

线程池的创建与销毁工作主要由三个参数决定：

+ corePoolSize：核心线程数。
+ maximumPoolSize：最大线程数。
+ keepAliveTime：存活时间。

**corePoolSize** 为线程池的基本大小，即使在没有任务时线程池保持的基本大小。在创建ThreadPoolExecutor 时，并不会立即启动线程，而是等到有任务提交时才会启动线程。除非调用 prestartAllCoreThreads 方法提前启动核心线程。

同时只有当工作队列已满时，才会创建超出这个数量的线程。开发人员有时会为了最终销毁工作线程时以免阻碍 JVM 的关闭而特地将 corePoolSize 设置为零。然而，如果线程池中没有使用 SynchronousQueue 作为工作队列时，这种方法将产生一些奇怪的行为。当 corePoolSize 设置为零时，只有当工作队列全部满时才会启动线程去执行任务，这种行为显然不符合我们的需求。

**maximumPoolSize** 为线程池可创建的最大线程数。

**keepAliveTime** 为线程的存活时间。如果某个线程的空闲时间超过了存活时间则被标记为可回收，并且当当前线程池大小超过 corePoolSize 时，这个线程就会被中止。

在 JDK 6 中，可以通过 allowCoreThreadTimeOut 方法将核心线程也包含在空闲销毁的范围内。如果希望这个线程池在没有任务时销毁所有线程，那么可以启动这个特性并将基本大小设置为零。

## 管理任务队列

如果一个线程池无限制的创建线程，那么将导致程序不稳定性，因此可以通过固定大小的线程池来解决这个问题。

然而，这个方案并不完整，因为在有限的线程池中会限制并发的任务数量。当新请求的到达速率超过线程池的处理速率时，新到的请求会被累积起来。在线程池中，累积的请求会在 Executor 管理的 Runnable 队列中等待，而不会同线程去竞争 CPU 资源。

一旦客户提交请求的速率超过了服务器的处理速率，那么仍可能会耗尽资源。因此，当任务持续高速到来，那么最终还需要抑制请求的到达率以避免耗尽内存。

> ⚠️ 注意：在耗尽内存之前，响应性能也会随着任务队列的增长而变得越来越糟。

ThreadPoolExecutor 允许提供一个 BlockingQueue 来保存等待执行的任务。基本的任务队列有 3 种：

+ 无界队列。
+ 有界队列。
+ 同步移交。

队列的选择根据线程池的其他参数有关，例如：线程池大小等。

**无界队列**

newFixedThreadPool 和 newSingleThreadPool 在默认情况下使用一个无界队列 LinkedBlockingQueue。当所有工作者线程都在忙碌时，那么任务将在队列中等待。当任务持续快速到达时，并且超过线程池的处理速率时，那么队列将无限制的增加。

> ⚠️ 注意：由于无界队列允许的队列长度为 Integer.MAX_VALUE，可能会堆积大量的任务，从而导致 OOM。

**有界队列**

相对于无界队列的一种更稳妥的资源管理策略，例如 ArrayBlockingQueue、有界的 LinkedBlockingQueue、PriorityBlockingQueue。

有界队列有助于避免资源耗尽的情况产生，但是需要提供饱和策略来处理队列已满时的任务。在线程池较小而队列较大，那么有助于减少内存使用量，降低 CPU 的使用率，同时还可以减少上下文的切换，但付出的代价是可能会限制吞吐量。

使用像 LinkedBlockingQueue 或 ArrayBlockingQueue 这类 FIFO 队列时，任务的执行顺序和到达顺序相同。而想控制任务的执行顺序，可以使用 PriorityBlockingQueue 来根据优先级安排任务。

**同步移交**

对于非常大或无界的线程可以通过 SynchronousQueue 来避免任务排队，以及直接将任务从生产者线程移交给工作者线程，跳过任务在队列中排队的过程。

SynchronousQueue 不是一个真正的队列，而是一种在线程之间进行移交的机制。移交过程为：

1. 将一个元素放入 SynchronousQueue 中。
2. 另一个线程等待接受这个元素。如果没有线程等待接收，并且线程池的当前大小小于最大值，那么 ThreadPoolExecutor 将创建新的线程，否则根据饱和策略，这个任务会被拒绝。

> 只有当线程池是无界的或者可以拒绝任务时，SynchronousQueue 才有使用价值。

## 饱和策略

当有界队列被填满后，饱和策略即发挥作用。ThreadPoolExecutor 的饱和策略可以通过调用 setRejectExecutionHandler 方法来修改。

JDK 默认提供了以下几种 RejectExecutionHandler 的实现：

+ AbortPolicy：默认策略，拒绝任务并抛出 RejectedException 异常。
+ CallerRunsPolicy：提交任务线程自行运行。
+ DiscardPolicy：抛弃任务且不会有反馈。
+ DiscardOldestPolicy：抛弃下一个被执行的任务，重试提交新任务。

> ⚠️ 注意：如果工作队列是一个优先级队列，则表示抛弃优先级最高的任务。因此最好不要将 DiscardOldestPolicy 策略和优先级队列一起使用。

该策略可以降低新任务的流量，但由于执行任务需要一定的时间，因此生产者线程在至少一段时间内无法从事本职工作，从而使得工作者线程有一定时间处理正在执行的任务。

## 线程工厂

每当线程池需要创建一个线程时，都是通过线程工厂方法来完成的。默认的线程工厂方法将创建一个新的、非守护的线程。

在许多情况下都需要定制的线程工程方法。例如：

+ 为线程池中的线程指定一个 UncaughtExceptionHandler。
+ 实例化定制 Thread 类用于执行调试信息的记录。
+ 为线程池中的线程取一个更有意义的名称。

如果需要利用安全策略来控制对某些特殊代码库的访问权限，可以通过 Executors 中的 privilegedThreadFactory 工厂方法来定制自己的线程工厂。通过这种方法创建出来的线程，将与创建 privilegedThreadFactory 的线程拥有相同的权限、AccessControlContext 和 contextClassLoader。

## 在调用构造函数后再定制 ThreadPoolExecutor

调用完 ThreadPoolExecutor 构造函数后，还是可以通过 setter 方法来修改大多数传递的构造函数参数。如果 Executor 是通过 Executors 中的某个工厂方法创建的，那么可以将结果的类型转换为 ThreadPoolExecutor 以使用 setter 方法：

```java
ExecutorService exec = Executors.newCachedThreadPool();
if (exec instanceof ThreadPoolExecutor) {
    ((ThreadPoolExecutor) exec).setCorePoolSize(10);
} else {
    throw new AssertionError("Oops, bad assumption");
}
```

Executors 中包含一个 unconfigurableExecutorService 工厂方法，该方法对一个现有的 ExecutorService 进行包装，使其只暴露 ExecutorService 的方法，因此不能对它进行配置。

newSingleThreadExecutor 方法即按这种方式封装的 ExecutorService，而不是最初的 ThreadPoolExecutor。防止在代码中增加单线程 Executor 的线程池大小，那么将破坏它的执行语义。因此如果将 ExecutorService 暴露给不信任的代码，又不希望对其进行修改，就可以通过 unconfigurableExecutorService 来包装它。