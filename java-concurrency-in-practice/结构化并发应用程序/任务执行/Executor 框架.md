# Executor 框架
#Java  #Multithreading 

Executor 接口虽然简单，但却为灵活且强大的异步任务执行框架提供了基础，该框架能支持多种不同类型的任务执行策略。它提供了一种标准的方法将任务的提交过程和执行过程解耦开来。同时接口的实现还提供了对生命周期的支持以及统计信息收集、应用程序管理机制和性能监视等机制。

Executor 基于生产者 - 消费者模式。提交任务的操作相当于生产者，执行任务的线程则相当于消费者。如果要在程序中实现一个生产者 - 消费者的设计，最简单的方式通常是使用 Executor。

例如，基于 Executor 实现的 Web 服务器：

```java
public class TaskExecutionWebServer {

    private static final int NTHREADS = 16;
    private static final Executor exec = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            exec.execute(() -> handleRequest(connection));
        }
    }

    private static void handleRequest(Socket connection) {
        // do something...
    }
}
```

## 执行策略

通过将任务的提交与执行解耦，从而无须太大的困难可以为某种类型的任务指定和修改执行策略。在执行策略中定义了任务的多个方面：

+ 在什么线程中执行任务？
+ 任务按照什么顺序执行？
+ 有多少个任务能并发执行？
+ 在队列中有多少个任务在等待执行？
+ 如果系统由于过载而需要拒绝一个任务，那么应该选择哪一个任务？如何通知应用程序有任务被拒绝？
+ 执行任务之前或之后，应该进行哪些动作？

各种执行策略都是一种资源管理工具，最佳策略取决于可用的计算资源以及对服务质量的需求。通过限制并发任务的数量，可以确保应用程序不会由于资源耗尽而失败，或者由于在稀缺资源上发生竞争而严重影响性能。

## 线程池

管理一组线程的资源池。线程池是与工作队列密切相关的，其中在工作队列中保存了所有等待执行的任务。工作线程的任务很简单，即从工作队列中获取一个任务，执行任务，最后返回线程池中等待下一个任务。

Java 类库提供了灵活的线程池以及一些有用的默认配置。可通过 Executors 中的静态工厂来创建线程池：

+ **newFixedThreadPool**：创建固定长度线程池，线程抛出异常后创建新的线程。
+ **newCacheThreadPool**：创建可变线程池，线程规模不受限制。
+ **newSingleThreadPool**：创建单线程的 Executor，线程抛出异常后创建新的线程。
+ **newScheduledThreadPool**：创建固定长度线程池，而且以延时或定时的方式执行任务。

通过使用 Executor 可以实现各种调优、管理、监视、记录日志、错误报告和其他功能，如果不使用任务执行框架，那么增加功能是非常困难的。

## Executor 生命周期

在创建 Executor 后，实现会创建线程来执行任务。而在关闭时，JVM 规定只有当所有非守护进程全部终止后才会退出，因此如果无法正常关闭 Executor 就无法正常关闭 JVM。

ExecutorService 扩展了 Executor 接口，提供了生命周期方法。ExecutorService 生命周期包含 3 种状态：

+ 运行：Executor 对象创建后处于该状态。
+ 关闭：对于平缓关闭不在接受新的任务，对于已提交任务等待执行完成。关闭后提交的任务由拒绝执行处理器来处理。
+ 已终止：当所有任务都完成后，ExecutorService 将转入终止状态。

ExecutorService  有两种关闭线程池的方式：shutdown 和 shutdownNow 方法。shutdown 方法将采用平缓关闭的方式：不再接收新的任务，同时等待已经提交的任务执行完成，包括还未开始执行的任务。shutdownNow 方法则采用粗暴的方式关闭：它将通过发送中断信号的方式尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务。

在 ExecutorService 关闭后提交的任务将由“拒绝处理器”来处理。

例如，支持关闭操作的服务器：

```java
public class LifecycleWebServer {

    private final ExecutorService executorService = Executors.newCachedThreadPool();

    public void start() throws IOException {
        ServerSocket serverSocket = new ServerSocket(80);
        while (!executorService.isShutdown()) {
            try {
                final Socket conn = serverSocket.accept();
                executorService.execute(() -> handleRequest(conn));
            } catch (RejectedExecutionException e) {
                if (!executorService.isShutdown()) {
                    System.out.println("task submission rejected");
                }
            }
        }
    }

    public void stop() {
        executorService.shutdown();
    }

    void handleRequest(Socket connection) {
        Request req = readRequest();
        if (isShutdownRequest(req)) {
            stop();
        } else {
            dispatchRequest(req);
        }
    }

    interface Request {
    }

    private void dispatchRequest(Request req) {

    }

    private Request readRequest() {
        return null;
    }

    private boolean isShutdownRequest(Request req) {
        return false;
    }
}
```

## 延迟任务与周期任务

JDK 早期的 Timer 类负责延迟任务以及周期任务，然而 Timer 类存在一些缺陷，应该考虑使用 ScheduledThreadPoolExecutor 来代替它。

Timer 类存在的问题;

1.  支持基于绝对时间而不是相对时间的调度机制，因此任务的执行对系统时钟很敏感。
2.  执行所有定时任务只会创建一个线程。如果某个任务执行时间过长会影响后续任务的定时精确。
3.  当任务抛出未检查异常，Timer 不会捕获而是抛出并终止定时线程，而且不会恢复线程后续的任务将都不会被执行，即使任务已经被调度。

使用 ScheduledThreadPoolExecutor，可以使用 DelayQueue 构建自己的调度服务。DelayQueue 管理着一组 Delayed 对象。每个 Delayed 对象都有一个相应的延迟时间：在 DelayQueue 中，只有某个元素逾期后，才能从 DelayQueue 中执行 take 操作。