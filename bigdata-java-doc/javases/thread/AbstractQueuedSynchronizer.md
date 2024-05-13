## AbstractQueuedSynchronizer

从java1.5开始提供了java.util.concurrent包，在此包中增加了在并发编程中很常用的工具类。其中AbstractQueuedSynchronizer是构建并发工具的基础。

### 队列同步器

AQS(AbstractQueuedSynchronizer)为队列同步器的缩写，是用来构建锁和其他同步组件的基础框架，使用一个int类型state来表示同步状态，同时里面通过一个双向链表组成的FIFO队列来完成资源的获取以及线程的排队工作。 队列同步器是实现锁的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

同步器的设计是基于模板方法模式的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

```java
队列同步器中的主要方法及可以重新的方法如下:

    //获取同步状态
    protected final int getState() {
        return state;
    }

    //设置同步状态
    protected final void setState(int newState) {
        state = newState;
    }

    //使用CAS设置当前状态，该方法能够保证状设置的原子性。
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
	
	//独占式获取同步状态，实现该方法需要查询当前的状态并判断同步状态是否符合预期，然后在进行cas设置同步状态
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    //独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
	
	//共享式获取同步状态，返回大于等于0的值，表示获取成功，反之获取失败
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //共享式释放同步状态
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
	
//队列同步器提供的模板方法


    //独占式获取同步状态,如果当前线程获取同步状态成功，则由该方法返回，否则将会进入人同步队列等待，该方法将会调用重写的tryAcquire(int arg)
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    //与acquire方法相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException并返回
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    //增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将会返回false,否则返回true
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

    //独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    //共享式获取同步状态，人如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以由多个线程获取到同步状态
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    //可以响应中断
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

    //增加超时限制
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }

    //共享式的释放同步状态
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```



### 同步队列

队列同步器依赖于内部的同步队列(FIFO)来完成同步状态的管理。其中同步队列中的节点（Node）用来保存线程的引用、获取同步状态、失败的线程引用、等待状态以及前驱和后继节点等。

```java
    //用来表示共享锁的标识
    static final Node SHARED = new Node();
    //用来表示独占锁的标识
    static final Node EXCLUSIVE = null;

    //值为1，由于在同步队列中等待的线程等待超时或者被中断,需要从同步去队列中取消等待，节点进行该状态将不会变化
    static final int CANCELLED =  1;
    //值为-1,后继节点的线程处于等待状态,而当前节点的线程如果释放了同步状态或者被取消,将会通知后继节点，使后继节点的线程得以运行
    static final int SIGNAL    = -1;
    //值为-2,节点在等待队列中,节点线程等待在Condition上,当其他线程对Condition调用了signal()方法后,该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
    static final int CONDITION = -2;
   //值为-3,表示下一次共享式同步状态获取将会无条件地传播下去
    static final int PROPAGATE = -3;

    //初始状态为0,等待状态即包含  CANCELLED、SIGNAL、CONDITION、PROPAGATE
    volatile int waitStatus;

    //前驱节点，当节点加入同步队列时被设置
    volatile Node prev;

    //后继节点
    volatile Node next;

    //获取同步状态的线程
    volatile Thread thread;

    //等待队列中的后继节点。如果当前节点时共享的，那么这个字段将是一个SHARED常量，也就是说节点类型(独占和共享)和等待队列中的后继节点公用同一个字段
    Node nextWaiter;
```

AQS类底层的数据结构使用双向链表,包含两个节点的引用，一个指向头结点，另一个指向尾结点。当一个线程获取到同步状态(或者锁),其他线程将无法获取同步状态，这时会构造一个Node节点加入到同步队列中，而这个过程必须保证线程安全。AQS是通过compareAndSetTail来保证安全的，底层调用的是unsafe.compareAndSwapObject的native方法。

### 独占式同步状态获取与释放

```java
//表示获取同步状态
public final void acquire(int arg) {
    //尝试获取同步状态，即获取锁
    if (!tryAcquire(arg) &&
    	//获取失败则加入同步队列，并尝试进行自旋获取同步状态
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //对线程进行中断操作
        selfInterrupt();
}
```

该方法可以获取同步状态,获取成功则返回,否则自旋获取锁，并且判断中断表示，如果中断标识为true，则设置线程中断。首先调用tryAcquire(arg)获取同步状态，该方法需要自定义实现，且保证线程安全。如果同步状态获取失败，则会进入到 addWaiter(Node mode)将同步节点加入到同步队列的尾部。然后调用acquireQueued(final Node node, int arg)方法，使该节点以死循环的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。

addWaiter方法用于节点的构造并加入队列

```java
private Node addWaiter(Node mode) {
	//根据当前线程创建一个独占的Node节点，mode为排他模式
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速入队，如果失败则降级至full enq，tail在AQS中表示同步队列队尾的属性，
    // 刚开始为null，所以进行enq（node）方法
    Node pred = tail;
    if (pred != null) {
        //设置前置节点
        node.prev = pred;
		//防止有其他线程修改tail，使用cas进行修改，保证原子性操作
        if (compareAndSetTail(pred, node)) {
		// 如果成功之后旧的tail 的next指针再指向新的tail，成为双向链表
            pred.next = node;
            return node;
        }
    }
	//如果队列为null或者cas设置新的tail失败,则进行enq方法进行设置尾结点
    enq(node);
    return node;
}

    private Node enq(final Node node) {
	//死循环，多次尝试直到成功为
    for (;;) {
		//队列为空，则首节点和尾节点都为空
        Node t = tail;
        if (t == null) { // Must initialize
			// 设置头结点，如果失败则存在竞争，留至下一轮循环
            if (compareAndSetHead(new Node()))
				//此时队列中只有一个节点，即首尾节点相同
                tail = head;
        } else {
			// 进行第二次循环时，tail不为null，进入else区域，将当前线程的
            // Node节点的prev指向tail，然后使用cas将tail指向Node，
            // 这部分代码和addWaiter代码一样，将当前节点添加到队列
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中（并会阻塞节点的线程）

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //中断标志
            boolean interrupted = false;
            //进行自旋获取节点状态
            for (;;) {
                // 获取prev节点，若为null即刻抛出NullPointException
                final Node p = node.predecessor();
                // 如果前驱为head才有资格进行锁的争夺
                if (p == head && tryAcquire(arg)) {
                    // 获取锁成功后就不需要进行同步操作了，
                    // 获取锁成功的线程为新的head节点
                    setHead(node);
                    // 凡是head节点，head.thread与head.prev永远为null，
                    // 但是head.next不为null
                    p.next = null; // help GC
                    // 获取锁成功
                    failed = false;
                    return interrupted;
                }
                // 如果获取锁失败，则根据节点的waitStatus决定是否需要阻塞线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 若前面为true，则执行阻塞，待下次唤醒的时候检测中断的标志
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 如果抛出异常则取消锁的获取，进行出队（sync queue）操作
            if (failed)
                cancelAcquire(node);
        }
    }
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	//前驱节点的状态
    int ws = pred.waitStatus;

    if (ws == Node.SIGNAL)
        //状态为signal，表示当前线程处于等待状态，直接放回true
        return true;
		//前驱节点状态 > 0 ，则为Cancelled,表明该节点已经超时或者被中断了，需要从同步队列中取消
    if (ws > 0) {
  
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
     //前驱节点状态为Condition、propagate
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

检查当前线程是否需要阻塞，即如果当前线程的前驱节点状态为SINNAL，则表明当前线程需要被阻塞，调用unpark()方法唤醒，直接返回true，当前线程阻塞。如果当前线程的前驱节点状态为CANCELLED（ws > 0），则表明该线程的前驱节点已经等待超时或者被中断了，则需要从CLH队列中将该前驱节点删除掉，直到回溯到前驱节点状态 <= 0 ，返回false。如果前驱节点非SINNAL，非CANCELLED，则通过CAS的方式将其前驱节点设置为SINNAL，返回false

![cb919752dbf611dda3b8f4f6822d962](.\pic\cb919752dbf611dda3b8f4f6822d962.png)

当前线程获取同步状态并执行了相应逻辑之后，就需要释放同步状态，使得后续节点能够继续获取同步状态。通过调用同步器的release(int arg)方法可以释放同步状态，该方法在释放了同步状态之后，会唤醒其后继节点。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //唤醒后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

总结：在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点。

唤醒后继节点

```java
private void unparkSuccessor(Node node) {   
 
    int ws = node.waitStatus;
    // 如果node的waitStatus<0 则使用CAS将等待状态改为0，即初始状态（因为下面马上要将node的后继节点唤醒）
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
 
    Node s = node.next; // 定义s为node的后继节点
    // 如果s为null或者waitStatus为CANCELLED
    if (s == null || s.waitStatus > 0) { 
        s = null;   // 直接将s赋值为null
        // 并从尾部向前遍历以找到实际未取消的后继节点（离node最近），
        // 这里的意思是将node之后的空节点或waitStatus=CANCELLED的节点也
        // 一并去掉，直接唤醒node之后waitStatus不为CANCELLED的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 如果s不为null，则唤醒s节点
        LockSupport.unpark(s.thread);   
}
```



### 共享式同步状态获取与释放

共享式获取与独占式获取最主要的区别在于同一时刻能否有多个线程同时获取到同步状态.通过调用同步器的acquireShared(int arg)方法可以共享式地获取同步状态.

```java
    public final void acquireShared(int arg) {
		//小于0表示没有获取同步状态
        if (tryAcquireShared(arg) < 0)
			//进行自旋
            doAcquireShared(arg);
    }
	
	    private void doAcquireShared(int arg) {
			//构建节点
        final Node node = addWaiter(Node.SHARED);
		
        boolean failed = true;
        try {
			// 线程中断标志
            boolean interrupted = false;
            for (;;) {
				// 获取node节点的前驱节点
                final Node p = node.predecessor();
				// 如果node的前驱节点是头结点
                if (p == head) {
                    int r = tryAcquireShared(arg);
					// 同步状态获取成功
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
				// 同步状态获取失败
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态，tryAcquireShared(int arg)方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。可以看到，在doAcquireShared(int arg)方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出

与独占式一样，共享式获取也需要释放同步状态，通过调用releaseShared(int arg)方法可以释放同步状态

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点。对于能够支持多个线程同时访问的并发组件（比如Semaphore），它和独占式主要区别在于tryReleaseShared(int arg)方法必须确保同步状态（或者资源数）线程安全释放，一般是通过循环和CAS来保证的，因为释放同步状态的操作会同时来自多个线程。







