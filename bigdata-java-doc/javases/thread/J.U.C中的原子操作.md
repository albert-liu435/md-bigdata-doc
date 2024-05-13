J.U.C中的原子操作

JUC中的原子类都是都是依靠**volatile**、**CAS**、**Unsafe**类配合来实现的，JUC中对原子操作提供了强大的支持，这些类位于**java.util.concurrent.atomic**包中。

#### 更新原子的基本类型

AtomicBoolean:原子更新布尔类型

AtomicInteger:原子更新整型

AtomicLon:原子更新长整型

AtomicInteger 类常用方法

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //value属性在AtomicInteger中的偏移量，通过这个偏移量可以快速定位到value字段，这个是实现AtomicInteger的关键。
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

   //使用volatile修饰，可以确保value在多线程中的可见性
    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    //获取当前的值
    public final int get() {
        return value;
    }

    //设置指定的值
    public final void set(int newValue) {
        value = newValue;
    }

    //最终会设置成 newValue,使用lazySet设置值后，可能导致其他线程在小段时间内还是可以读到旧的值。
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    //以原子方式设置为newValue的值，并返回旧值
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    //如果输入的数值等于预期值，则以原子的方式将该值设置为输入的值
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    //
    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

   //以原子的方式将当前值加1，并返回自增前的值
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    //以原子的方式进行减1,
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }

    //以原子的方式加上指定的值
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    //以原子的方式将当前值加1，并返回自增后的值
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

    //
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }

    //以原子方式将输入的数值与实例中的值(AtomicInteger里的value)进行相加，并返回结果
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }

    
    public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }

    
    public final int updateAndGet(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return next;
    }

    
    public final int getAndAccumulate(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));
        return prev;
    }

    
    public final int accumulateAndGet(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));
        return next;
    }

    
    public String toString() {
        return Integer.toString(get());
    }

    
    public int intValue() {
        return get();
    }

    
    public long longValue() {
        return (long)get();
    }

    
    public float floatValue() {
        return (float)get();
    }	
	
    public double doubleValue() {
        return (double)get();
    }

}

```

AtomicBoolea，AtomicLon里面的方法都类似

#### 原子更新数组

AtomicIntegerArray:原子更新整型数组里面的元素
AtomicLongArray:原子更新长整型数组里的元素
AtomicReferenceArray:原子更新引用类型数组里的元素

AtomicIntegerArray在操作数组的时候，首先会对数组进行复制一份，然后再在AtomicIntegerArray内部进行操作，所以不会影响原有数组的值的更改。

```java
public class AtomicIntegerArrayTest {

    static int[]              value = new int[] { 1, 2 };

    static AtomicIntegerArray ai    = new AtomicIntegerArray(value);

    public static void main(String[] args) {
        ai.getAndSet(0, 3);
        //输出3
        System.out.println(ai.get(0));
        //输出1
        System.out.println(value[0]);
    }

}
```

#### 原子更新引用类型

AtomicReference:原子更新引用类型

AtomicReferenceFieldUpdater:原子更新引用类型里面的字段

AtomicMarkableReference：原子更新带有标记位的引用类。可以原子更新一个布尔类型的标记位 和引用类型。

#### 原子更新字段类型

AtomicIntegerFieldUpdater：原子更新整形字段的值
AtomicLongFieldUpdater：原子更新长整形字段的值
AtomicReferenceFieldUpdater ：原子更新应用类型字段的值

