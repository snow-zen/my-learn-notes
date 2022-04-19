# 扩展 ThreadPoolExecutor
#Java #Multithreading 

ThreadPoolExecutor 是可扩展的，它提供了几个可以在子类中改写的方法：

+ beforeExecute
+ afterExecute
+ terminated

beforeExecute 方法在任务执行前调用。如果方法抛出一个 RuntimeException，那么任务将不被执行，并且 afterExecute 也不会被调用。

afterExecute 方法在任务执行后被调用。如果任务完成后带有一个 Error，那么 afterExecute 就不会被调用。

terminated 方法在线程池关闭时被调用。

例如，给线程池添加执行计时信息：

```java
public class TimingThreadPool extends ThreadPoolExecutor {

    public TimingThreadPool() {
        super(1, 1, 0L, TimeUnit.SECONDS, null);
    }

    private final ThreadLocal<Long> startTime = new ThreadLocal<>();
    private final Logger log = Logger.getLogger("TimingThreadPool");
    private final AtomicLong numTasks = new AtomicLong();
    private final AtomicLong totalTime = new AtomicLong();


    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        log.fine(String.format("Thread %s: start %s", t, r));
        startTime.set(System.nanoTime());
    }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        try {
            long endTime = System.nanoTime();
            long taskTime = endTime - startTime.get();
            numTasks.incrementAndGet();
            totalTime.addAndGet(taskTime);
            log.fine(String.format("Thread %s: end %s, time = %dns", t, r, taskTime));
        } finally {
            startTime.remove();
            super.afterExecute(r, t);
        }
    }

    @Override
    protected void terminated() {
        try {
            log.info(String.format("Terminated: avg time = %dns", totalTime.get() / numTasks.get()));
        } finally {
            super.terminated();
        }
    }
}
```