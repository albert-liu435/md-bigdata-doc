java线程池

由于线程和数据库连接这些资源都是非常宝贵的的资源，每次需要的时候创建，不需要的时候销毁，是非常消耗资源的,所以需要使用线程池。使用线程池主要有三个好处,降低资源消耗，提高响应速度，提高线程的可管理性。

### 线程池实现原理

当向线程池提交一个任务时,线程池的处理流程如下:

1、线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
2、线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
3、线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

![42b7e811b19684f07d6f5ca6b5ef89c](.\pic\42b7e811b19684f07d6f5ca6b5ef89c.png)

### java线程池框架

![1066844-20200322174614850-1861868203](.\pic\1066844-20200322174614850-1861868203.png)

Executor接口

```java
public interface Executor {

    //接受一个Runable实例，他来执行一个任务，任务即实现一个Runable接口的类。
    void execute(Runnable command);
}
```

ExcutorService

ExecutorService继承于Executor接口，他提供了更为丰富的线程实现方法，比如ExecutorService提供关闭自己的方法，以及为跟踪一个或多个异步任务执行状况而生成Future的方法。

```java
public interface ExecutorService extends Executor {

    //关闭线程池，首先让 ExecutorService 停止接受新任务，并在所有正在运行的线程完成当前工作后关闭。
    void shutdown();

    //首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表
    List<Runnable> shutdownNow();

    //如果此执行程序已关闭，则返回 true。
    boolean isShutdown();

    //如果关闭后所有任务都已完成，则返回 true
    boolean isTerminated();

    //请求关闭、发生超时或者当前线程中断，无论哪一个首先发生之后，都将导致阻塞，直到所有任务完成执行
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    //提交一个有返回值的任务，返回值的结果放到Future
    <T> Future<T> submit(Callable<T> task);

    //提交一个有返回值的任务，返回值的结果放到Future
    <T> Future<T> submit(Runnable task, T result);

    //提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
    Future<?> submit(Runnable task);

    //执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    //执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    //执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    //执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，可以在给定的延迟后运行命令，或者定期执行命令。ScheduledThreadPoolExecutor比Timer更灵活，功能更强大。

ThreadPoolExecutor

ThreadPoolExecutor是线程池的核心实现类，用来执行被提交的任务

Future&Callable

Future接口定义了操作异步异步任务执行一些方法，如获取异步任务的执行结果、取消任务的执行、判断任务是否被取消、判断任务执行是否完毕等

Callable接口中定义了需要有返回的任务需要实现的方法。

Executors

Executors类，提供了一系列工厂方法用于创建线程池，返回的线程池都实现了ExecutorService接口Executors类，提供了一系列工厂方法用于创建线程池，返回的线程池都实现了ExecutorService接口.常用方法如下：

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

