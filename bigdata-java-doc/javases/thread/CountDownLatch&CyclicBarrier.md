CountDownLatch&CyclicBarrier

jdk中的Thread类中提供了join方法,拥有表示让当前线程等待join线程执行结束后再执行当前的线程的逻辑。可以看一下join的源码，内部其实是在synchronized方法中调用了线程的wait方法，最后被调用的线程执行完毕之后，由jvm自动调用其notifyAll()方法，唤醒所有等待中的线程。在JDK 1.5之后的并发包中提供的CountDownLatch也可以实现join的功能，并且比join的功能更多.

### CountDownLatch

CountDownLatch是通过一个计数器来实现的，当我们在new 一个CountDownLatch对象的时候需要带入该计数器值，该值就表示了线程的数量。每当一个线程完成自己的任务后，计数器的值就会减1。当计数器的值变为0时，就表示所有的线程均已经完成了任务，然后就可以恢复等待的线程继续执行了。

```java
public class CountDownLatch {
	//内部同步队列器
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }
		
		//获取共享同步状态
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    //传入计数器，该count值必须大于0
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    //调用await()会让当前线程等待，直到计数器为0的时候，方法才会返回，此方法会响应线程中断操作。
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //限时等待，在超时之前，计数器变为了0，方法返回true，否者直到超时，返回false，此方法会响应线程中断操作。
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //用于让计数器减一
    public void countDown() {
        sync.releaseShared(1);
    }

    //返回当前的值
    public long getCount() {
        return sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```

以老师在教室等待学生上课为例，当所有的学生都进入教室之后开始上课，示例代码如下：

```java
public class CountDownLatchHouse {


    private static CountDownLatch countDownLatch = new CountDownLatch(5);
    
    static class TeacherThread extends Thread{
        @Override
        public void run() {
            System.out.println("学校老是在等待学生进入教师，总共有" + countDownLatch.getCount() + "个人上课");
            try {
                //老师等待
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("所有人都已经到齐，开始上课");
        }
    }

    //学生到达教室
    static class StudentThread  extends Thread{



        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "进入教室");
            //学生到达教室 count - 1
            countDownLatch.countDown();
        }
    }

    public static void main(String[] args){
        //老师线程启动
        new TeacherThread().start();

        for(long i = 0,j = countDownLatch.getCount() ; i < j ; i++){
            new StudentThread().start();
        }
    }
}
```

### CyclicBarrier

CyclicBarrier 让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。CyclicBarrier的内部是使用重入锁ReentrantLock和Condition。

```java

public class CyclicBarrier {

    private static class Generation {
        boolean broken = false;
    }

    //锁,用来守卫屏障
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    //拦截线程的数量
    private final int parties;
    //用于在线程到达屏障时，优先执行
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    //等待线程的数量
    private int count;

    //
    private void nextGeneration() {
        // 唤醒所有的线程
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

    
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    //每调用一次await()方法都将使阻塞的线程数+1，只有阻塞的线程数达到设定值时屏障才会打开，允许阻塞的所有线程继续执行
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
		//获取锁
        lock.lock();
        try {
			
            final Generation g = generation;
			
            if (g.broken)
                throw new BrokenBarrierException();

			//如果线程中断，终止CyclicBarrier
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

			//进来一个线程 count - 1
            int index = --count;
            if (index == 0) {  // tripped
			////count == 0 表示所有线程均已到位，触发Runnable任务
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
						//这里调用的是run方法
                        command.run();
                    ranAction = true;
					//唤醒所有等待线程，并更新generation
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
					//如果不是超时等待，则调用Condition.await()方法等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
						//超时等待，调用Condition.awaitNanos()方法等待
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
			 //释放锁
            lock.unlock();
        }
    }

    //创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，并在启动 barrier 时执行给定的屏障操作，该操作由最后一个进入 barrier 的线程执行
	//barrierAction 为CyclicBarrier接收的Runnable命令，用于在线程到达屏障时，优先执行barrierAction ，用于处理更加复杂的业务场景
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    //创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动
    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    /**
     * Returns the number of parties required to trip this barrier.
     *
     * @return the number of parties required to trip this barrier
     */
    public int getParties() {
        return parties;
    }

    //
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
			//不超时等待
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    //超时等待
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    //
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    //
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    //
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}

```

示例

```java
public class CyclicBarrierTest {

    private static CyclicBarrier cyclicBarrier;

    static class CyclicBarrierThread extends Thread{
        public void run() {
            System.out.println(Thread.currentThread().getName() + "到了");
            //等待
            try {
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        cyclicBarrier = new CyclicBarrier(5, new Runnable() {
            @Override
            public void run() {
                System.out.println("学生到齐了，开始上课");
            }
        });

        for(int i = 0 ; i < 5 ; i++){
            Thread.sleep(1000);
            new CyclicBarrierThread().start();
        }
    }
}
```

CountDownLatch与CyclicBarrier

ountDownLatch: 一个线程(或者多个)， 等待另外N个线程完成某个事情之后才能执行。CyclicBrrier: N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。

CountdownLatch利用继承AQS的共享锁来进行线程的通知,利用CAS来进行--,而CyclicBarrier则利用ReentrantLock的Condition来阻塞和通知线程

### Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，即可以控制访问资源的线程数量。它通过协调各个线程，以保证合理的使用公共资源。Semaphore可以用于做流量控制，特别是公用资源有限的应用场景.Semaphore通过构造许可证的方式控制访问资源的线程数，其本质上类似一个多线程访问共享锁的模式。

Semaphore方法

```java
public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
   
    private final Sync sync;

    //同步队列的实现
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    //非公平
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    //公平
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    //传入许可证数来构造信号量，默认为非公平的信号量
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    //当fair等于true时，创建具有给定许可数的计数信号量并设置为公平信号量
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

    //获取一个许可，此方法会响应线程中断，表示调用线程的interrupt方法，会使该方法抛出InterruptedException异常
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }

    //尝试获取1个许可，不管是否能够获取成功，都立即返回，true表示获取成功，false表示获取失败
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    //
    public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //
    public void release() {
        sync.releaseShared(1);
    }

    //获取指定数量的许可
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    //获取指定数量的许可，不会响应线程中断
    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }

    //
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

    //
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }

    //
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

    //此信号量中当前可用的许可证数
    public int availablePermits() {
        return sync.getPermits();
    }

    /**
     * Acquires and returns all permits that are immediately available.
     *
     * @return the number of permits acquired
     */
    public int drainPermits() {
        return sync.drainPermits();
    }

    //减少reduction个许可证
    protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }

    public boolean isFair() {
        return sync instanceof FairSync;
    }

    //是否有线程正在等待获取许可证
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    //正在等待获取许可证的线程数
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    //所有等待获取许可证的线程集合
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    public String toString() {
        return super.toString() + "[Permits = " + sync.getPermits() + "]";
    }
}

```

以停车场为例，假设有3个停车位，当停车场满了以后，只有有车从停车场开出来另一辆车才能进入停车场。

```java
public class SemaphoreTest {


    static class Parking{
        //信号量
        private Semaphore semaphore;

        Parking(int count){
            semaphore = new Semaphore(count);
        }

        public void park(){
            //用于标识信号量是否获取成功，否则有可能导致许可数量凭空增加
            boolean flag=false;
            try {
                //获取信号量
                semaphore.acquire();
                flag=true;
                long time = (long) (Math.random() * 10);
                System.out.println(Thread.currentThread().getName() + "进入停车场，停车" + time + "秒..." );
                Thread.sleep(time);
                System.out.println(Thread.currentThread().getName() + "开出停车场...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {

                if(flag){
                    semaphore.release();
                }


            }
        }
    }

    static class Car extends Thread {
        Parking parking ;

        Car(Parking parking){
            this.parking = parking;
        }

        @Override
        public void run() {
            parking.park();     //进入停车场
        }
    }

    public static void main(String[] args){
        Parking parking = new Parking(3);

        for(int i = 0 ; i < 5 ; i++){
            new Car(parking).start();
        }
    }
}
```

