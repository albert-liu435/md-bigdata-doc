java线程

java中的线程可以理解为人，单线程理解为一个人，多线程可以理解为多个人，一般情况下，一项任务分配给多个人要比分配给一个人花费的时间短。所以java的多线程就是为了工作起来更快更省时间。

### java线程状态

新建(New)：当程序使用new创建一个线程之后，该线程就处于新建状态，此时还没有调用start()方法。

就绪状态(Runnable)：当线程对象调用了start()方法之后，该线程处于就绪状态。

运行状态(Running)：如果处于就绪状态的线程获得了CPU,开始执行run()方法的线程执行体，则该线程处于运行状态。

阻塞状态(Blocked)：阻塞状态是指线程因为某种原因放弃了CPU使用权，也即让出了CPU timeslice,暂时停止运行。直到线程进入可运行状态，才有机会再次获取CPU timeslice转到运行状态。阻塞分下面三种情况：

1、等待阻塞(o.wait->等待队列):运行的线程执行o.wait()方法，JVM会把该线程放入等待队列中

2、同步阻塞(lock)：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。

3、其他阻塞(sleep/join):运行的线程执行sleep或join方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程或者超时、或者I/O处理完毕时，线程重新转入可运行状态。

线程死亡(Dead)：线程死亡主要包括下面三种方式：

1、正常结束：run()或call()方法执行完成，线程正常结束。

2、异常结束：线程抛出异常Exception或者Error。

3、调用stop:直接调用该线程的stop()方法来结束该线程。

![1066844-20200322174417330-688259916](.\pic\1066844-20200322174417330-688259916.jpg)

线程创建之后，调用start()方法开始运行。当线程执行wait()方法之后，线程进入等待状态。进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而超时等待状态相当于在等待状态的基础上增加了超时限制，也就是超时时间到达时将会返回到运行状态。当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到阻塞状态。线程在执行Runnable的run()方法之后将会进入到终止状态。

### java启动main线程

在windows环境下运行如下代码，则打印出如下信息

```java
    public static void main(String[] args) {
        // 获取Java线程管理MXBean
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 不需要获取同步的monitor和synchronizer信息，仅仅获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        // 遍历线程信息，仅打印线程ID和线程名称信息
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.getThreadName());
        }
    }
```

```java
//打印出的信息
[6] Monitor Ctrl-Break
[5] Attach Listener
[4] Signal Dispatcher
[3] Finalizer
[2] Reference Handler
[1] main
```

说明启动一个java进程应用程序，启动了如上的六个线程。其中

Attach Listener：负责接收到外部的命令，而对该命令进行执行的并且吧结果返回给发送者。如：java -version、jmap、jstack等等

Signal Dispatcher：前面我们提到第一个Attach Listener线程的职责是接收外部jvm命令，当命令接收成功后，会交给signal dispather线程去进行分发到各个不同的模块处理命令，并且返回处理结果。

Finalizer：在main线程之后创建的，其优先级为10，主要用于在垃圾收集前，调用对象的finalize()方法。

Reference Handler：VM在创建main线程后就创建Reference Handler线程，其优先级最高，为10，它主要用于处理引用对象本身（软引用、弱引用、虚引用）的垃圾回收问题。

### 线程创建的三种方式

继承Thread类

```java
public class MyThread extends Thread {

    private String name;

    public MyThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println("执行 " + i + name);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        MyThread thread1 = new MyThread("thread1");
        thread1.start();

        MyThread thread2 = new MyThread("thread2");
        thread2.start();

    }

}
```

实现Ruunnable接口

```java
public class MyRunnable implements Runnable {

    private String name;

    public MyRunnable(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println("执行 " + i + name);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        MyRunnable runnable1 = new MyRunnable("runnable1");
        Thread thread1 = new Thread(runnable1);
        thread1.start();

        MyRunnable runnable2 = new MyRunnable("runnable2");
        Thread thread2 = new Thread(runnable2);
        thread2.start();


    }

}
```

实现Callable接口

```java
public class MyCallable implements Callable {
    private String name;

    public MyCallable(String name) {
        this.name = name;
    }

    @Override
    public Object call() throws Exception {
        for (int i = 0; i < 100; i++) {
            System.out.println("执行 线程" + i + name);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return "task";
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        MyCallable myCallable = new MyCallable("callabe");
        FutureTask futureTask = new FutureTask(myCallable);
        Thread thread = new Thread(futureTask);
        thread.start();

        System.out.println(futureTask.get());
        System.out.println("执行主线程");

    }

}

```

无论这三种方式如何构建，最终在创建线程的时候都会调用Thread类中的init方法

```java
  private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }
		//线程名称
        this.name = name;
		//当前线程就是该线程的父线程
        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
		
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();
		//该线程所属的线程组
        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        //设置线程ID
        tid = nextThreadID();
    }
```

参考：[Java 启动一个main程序时，有多少个线程](https://blog.csdn.net/chenxi004/article/details/104972979)