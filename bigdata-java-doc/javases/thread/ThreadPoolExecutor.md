ThreadPoolExecutor

### 构造方法

通过构造方法创建线程池

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

**corePoolSize**(核心线程数量):当提交一个任务到线程池时,线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程。等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。
**maximumPoolSize**:线程池中允许的自大线程数。如果队列满了，并且已经创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。如果我们使用了无界队列，那么所有的任务会加入队列，这个参数就没有什么效果了
**keepAliveTime**:线程空闲的时间。线程的创建和销毁是需要代价的。线程执行完任务后不会立即销毁，而是继续存活一段时间：keepAliveTime。默认情况下，该参数只有在线程数大于corePoolSize时才会生效。
**unit**：keepAliveTIme的时间单位，可以选择的单位有天、小时、分钟、毫秒、微妙、千分之一毫秒和纳秒
**workQueue**:用来保存等待执行的任务的阻塞队列，等待的任务必须实现Runnable接口.主要有四种
	ArrayBlockingQueue：基于数组结构的有界阻塞队列，FIFO。
	LinkedBlockingQueue：基于链表结构的有界阻塞队列，FIFO。
	SynchronousQueue：不存储元素的阻塞队列，每个插入操作都必须等待一个移出操作，
	PriorityBlockingQueue：具有优先界别的阻塞队列。
**threadFactory**：线程池中创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字
**handler**：饱和策略，当线程池无法处理新来的任务了，那么需要提供一种策略处理提交的新任务

AbortPolicy：直接抛出异常。
CallerRunsPolicy：只用调用者所在线程来运行任务。
DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
DiscardPolicy：不处理，丢弃掉

### ThreadPoolExecutor示例

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) throws InterruptedException {
         ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(3,5,10,TimeUnit.SECONDS,new ArrayBlockingQueue<Runnable>(10),Executors.defaultThreadFactory(),new ThreadPoolExecutor.AbortPolicy());


//        List<Callable<String>> list=new ArrayList<>();
//        //当i=100的时候，超过队列的数量10，所以会报出 RejectedExecutionException
//        for(int i=0;i<10;i++){
//            list.add(new Callable<String>() {
//                @Override
//                public String call() throws Exception {
//                    System.out.println(Math.random());
//                    return "string";
//                }
//            });
//        }
//        threadPoolExecutor.invokeAll(list);
        //当i=100时，也会报RejectedExecutionException
        for (int i = 0; i < 100; i++) {
            int j = i;
            String taskName = "任务" + j;
            threadPoolExecutor.execute(() -> {
                //模拟任务内部处理耗时
                try {
                    TimeUnit.SECONDS.sleep(j);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + taskName + "处理完毕");
            });
        }
        //关闭线程池
        threadPoolExecutor.shutdown();

    }
}
```

 threadPoolExecutor.execute(Runnable command)方法

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        //如果当前运行的线程数少于corePoolSize，则创建新线程来执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //线程池处于运行状态，则尝试加入队列，加入队列成功后再次进行check,如果加入失败，则执行下面操作
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果线程池不是RUNNING状态或者加入阻塞队列失败，则尝试创建新线程直到maxPoolSize，如果失败，则调用reject()方法运行相应的拒绝策略。
        else if (!addWorker(command, false))
            reject(command);
    }
```

### ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor 继承自ThreadPoolExecutor，其中的构造方法如下

```java
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }
```

DelayedWorkQueue是一个无界队列，所以ThreadPoolExecutor的maximumPoolSize在ScheduledThreadPoolExecutor中没有什么意义

示例

```java
public class ScheduledThreadPoolExecutorDemo {

    public static void main(String[] args) {

        Date date = new Date();
        System.out.println(date);

        ScheduledThreadPoolExecutor scheduledThreadPoolExecutor=new ScheduledThreadPoolExecutor(10);


       // scheduledThreadPoolExecutor.sche
        scheduledThreadPoolExecutor.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                Date date = new Date();
                System.out.println("hello"+ date);
            }
        },10L,10, TimeUnit.SECONDS);

       // scheduledThreadPoolExecutor.scheduleWithFixedDelay()
    }
}
```

