java的线程池

由于线程和数据库连接这些资源都是非常宝贵的的资源，每次需要的时候创建，不需要的时候销毁，是非常消耗资源的。为在java在java.util.concurrent包下为我们提供了线程池供我们使用。

java线程池的顶级接口是Executor，如下实现图

![Executor](.\pic\Executor.png)

常采用Executors来创建线程池，其中最常用如下：

```java
public class Executors {


    /**
     * 创建一个根据需要创建新线程的线程池,当调用execute将重用以前构造出来的线程，
     * 如果现有的线程没有可用的,将会重新创建新的线程，并添加到线程池中，
     *如果线程在60s没有被使用，将会从线程池中移除。如果长时间保持空闲，该线程池不会占用任何资源
     *
     * @return the newly created thread pool
     */
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    
    /**
     * 创建固定数量线程的线程池，如果线程池中没有足够的数量的线程来执行任务，
     * 则任务将会在队列中等待，直到有线程来执行它。除非线程被显示的关闭，否则线程池中的线程将会一直存在
     *
     * @param nThreads the number of threads in the pool
     * @param threadFactory the factory to use when creating new threads
     * @return the newly created thread pool
     * @throws NullPointerException if threadFactory is null
     * @throws IllegalArgumentException if {@code nThreads <= 0}
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
    
    /**
     * 创建一个线程池，可以安排在给定延迟命令或者定期地执行
     * @param corePoolSize the number of threads to keep in the pool,
     * even if they are idle
     * @return a newly created scheduled thread pool
     * @throws IllegalArgumentException if {@code corePoolSize < 0}
     */
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    
    /**
     * 创建一个线程的线程池，当这个线程池中的线程死后，或者发生异常，重新启动一个线程来代替原来的线程
     *
     * @return the newly created single-threaded Executor
     */
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }	

}
```

