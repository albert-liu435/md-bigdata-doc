## ReenTrantLock

重入锁，表示支持重新进入的锁，也就是说如果当前线程t1通过调用lock方法获取了锁之后，再次调用lock，是不会再阻塞去获取锁的，直接增加重试次数就行了。同时该锁还支持获取锁时的公平和非公平性的选择，默认情况下为 非公平锁。

### 锁的重入

锁的重入机制是当线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再
次成功获取。

线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。



当调用ReentrantLock.lock()方法时，会调用ReentrantLock里面的sync.lock()方法，sync有两个内部实现类，分别为FairSyn(公平锁)和NonfairSync(非公平锁)。

### 非公平锁

```java
final void lock() {
    //用来进行获取锁，如果锁空闲就会获取到锁
    if (compareAndSetState(0, 1))
        // 用来保存当前占用同步状态的线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //尝试获取锁
        acquire(1);
}
```

compareAndSetState是通过cas算法来对state的值，如果CAS操作未能成功，说明state已经不为0，此时继续acquire(1)操作。

```
public final void acquire(int arg) {
	//尝试获取锁
    if (!tryAcquire(arg) &&
    	//获取锁失败后就会加入到同步队列中，并进行自旋获取锁
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

对与非公平锁来说，tryAcquire方法最终调用的是nonfairTryAcquire方法。

```java
final boolean nonfairTryAcquire(int acquires) {
    //当前线程
    final Thread current = Thread.currentThread();
    //state的值
    int c = getState();
    //首先判断状态是否为0，
    if (c == 0) {
        //运用cas算法进行加锁，这也是非公平锁的实现逻辑
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //判断加锁线程就是当前持有锁的线程时,会将state的值再次进行加1，这段逻辑也是ReenTrantLock可重入的原因
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

ReenTrantLock.unlock方法

释放锁的过程主要调用release方法

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

protected final boolean tryRelease(int releases) {
    //将获取锁的次数进行减1
    int c = getState() - releases;
    // 如果释放的线程和获取锁的线程不是同一个，抛出非法监视器状态异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //只有当最后的状态为0的情况下，才会释放锁成功
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

即：如果该锁被获取了n次，那么前(n-1)次tryRelease(int releases)方法必须返回false，而只有同
步状态完全释放了，才能返回true。可以看到，该方法将同步状态是否为0作为最终释放的条
件，当同步状态为0时，将占有线程设置为null，并返回true，表示释放成功。

### 公平锁

```java
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //判断是否有前驱节点
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平锁和非公平锁的逻辑中只是多了一个hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。











