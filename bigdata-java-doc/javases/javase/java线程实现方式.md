java线程实现方式

首先，单线程理解为一个人，多线程可以理解为多个人，一般情况下，一项任务分配给多个人要比分配给一个人花费的时间短。所以java的多线程就是为了工作起来更快更省时间。

java线程

java线程指程序运行过程中的一个线程实体，JVM允许一个应用并发执行多个线程，Hotspot JVM中的java线程与原生操作系统线程有着直接的映射关系。当线程本地存储、缓冲区分配、同步对象、栈、程序计数器等准备好以后，就会创建一个操作系统的原生线程。java线程结束，原生线程随之被回收。操作系统负责调度所有线程，并把他们分配到任何可用的CPU上，当原生系统初始化完毕，就会调用java线程的run()方法。当线程结束时，会释放原生线程和java线程的所有资源。

java线程创建的两种方式

#### 继承Thread类

启动线程的唯一方法就是通过Thread类的start()实例方法，start()方法是一个native方法，它将启动一个新线程，并执行run()方法

如下代码

```java
package com.rookie.bigdata.thread;


/**

 * @author rookie

 * @version 1.0

 * @date 2020/3/19 21:16
   */
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

#### 实现Runnable接口

另一种方式就是实现Runnable接口

```java
package com.rookie.bigdata.thread;

/**

 * @author rookie

 * @version 1.0

 * @date 2020/3/19 21:23
   */
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

#### 实现Callable接口

实现Callable接口功能要比上面两种类型更强大，如下

```java
package com.rookie.bigdata.thread;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**

 * @author rookie

 * @version 1.0

 * @date 2020/3/19 21:56
   */
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
       MyCallable myCallable=new MyCallable("callabe");
       FutureTask futureTask=new FutureTask(myCallable);
       Thread thread=new Thread(futureTask);
       thread.start();

       System.out.println(futureTask.get());
       System.out.println("执行主线程");

   }

}
```

采用实现Callable接口的方式时，如果在启动线程中调用futureTask.get()方法时，启动线程就会等待调用的线程执行完毕以后，启动线程才会往下执行其他的流程。

#### 线程的生命周期

当线程没创建启动以后，它既不是启动就马上进入了执行状态，也不是一直处于执行状态。在线程的生命周期中，它需要经过新建(New)、就绪(Runnable)、运行(Running)、阻塞（Blocked）和死亡(Dead) 5种状态。尤其当线程启动以后，不是一直占用CPU运行，CPU是在多条线程之间切换，于是线程状态也会多次在运行、阻塞之间切换

##### 新建(New)

当程序使用new创建一个线程之后，该线程就处于新建状态，

##### 就绪状态(Runnable)

当线程对象调用了start()方法之后，该线程处于就绪状态。

##### 运行状态(Running)

如果处于就绪状态的线程获得了CPU,开始执行run()方法的线程执行体，则该线程处于运行状态。

##### 阻塞状态(Blocked)

阻塞状态是指线程因为某种原因放弃了CPU使用权，也即让出了CPU timeslice,暂时停止运行。直到线程进入可运行状态，才有机会再次获取CPU timeslice转到运行状态。阻塞分下面三种情况：

1、等待阻塞(o.wait->等待队列):运行的线程执行o.wait()方法，JVM会把该线程放入等待队列中

2、同步阻塞(lock)：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。

3、其他阻塞(sleep/join):运行的线程执行sleep或join方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程或者超时、或者I/O处理完毕时，线程重新转入可运行状态。

##### 线程死亡(Dead)

线程死亡主要包括下面三种方式：

1、正常结束：run()或call()方法执行完成，线程正常结束。

2、异常结束：线程抛出异常Exception或者Error。

3、调用stop:直接调用该线程的stop()方法来结束该线程。

![thread](.\pic\thread.jpg)

相关源码参考： [https://github.com/albert-liu435/rookies-javases/tree/master/rookie-javase-thread]( https://github.com/albert-liu435/rookies-javases/tree/master/rookie-javase-thread ) 

