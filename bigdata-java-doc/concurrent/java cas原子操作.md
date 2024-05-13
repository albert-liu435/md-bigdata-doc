java cas原子操作
原子操是不可被中断的一个或一系列的操作，计算机处理器处理原子操作的方法如下
1、使用总线锁保证原子性，所谓总线锁就是使用处理器提供的一个LOCK＃信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存
2、使用缓存锁保证原子性，在同一时刻，我们只需保证对某个内存地址的操作是原子性即可

### cas操作

CAS,compare and swap的缩写，即判断内存中某个地址的值是否为预期值，如果是就改变成新值，整个过程具有原子性。 是一种无锁的非阻塞算法的实现

CAS操作包含三个操作数：内存位置(V)、预期原值(A))和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作

### java中的Unsafe

java中CAS的操作类为sun.misc.Unsafe,里面主要为下面的三个方法，这三个方法全部为native方法，即调用非java代码的接口底层采用C或者C++编写。我们暂时不需要了解底层只要知道这三个主要的方法的作用即可。

```java


package sun.misc;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.security.ProtectionDomain;
import sun.reflect.CallerSensitive;
import sun.reflect.Reflection;

public final class Unsafe {
    private static final Unsafe theUnsafe;
    public static final int INVALID_FIELD_OFFSET = -1;
    public static final int ARRAY_BOOLEAN_BASE_OFFSET;
    public static final int ARRAY_BYTE_BASE_OFFSET;
    public static final int ARRAY_SHORT_BASE_OFFSET;
    public static final int ARRAY_CHAR_BASE_OFFSET;
    public static final int ARRAY_INT_BASE_OFFSET;
    public static final int ARRAY_LONG_BASE_OFFSET;
    public static final int ARRAY_FLOAT_BASE_OFFSET;
    public static final int ARRAY_DOUBLE_BASE_OFFSET;
    public static final int ARRAY_OBJECT_BASE_OFFSET;
    public static final int ARRAY_BOOLEAN_INDEX_SCALE;
    public static final int ARRAY_BYTE_INDEX_SCALE;
    public static final int ARRAY_SHORT_INDEX_SCALE;
    public static final int ARRAY_CHAR_INDEX_SCALE;
    public static final int ARRAY_INT_INDEX_SCALE;
    public static final int ARRAY_LONG_INDEX_SCALE;
    public static final int ARRAY_FLOAT_INDEX_SCALE;
    public static final int ARRAY_DOUBLE_INDEX_SCALE;
    public static final int ARRAY_OBJECT_INDEX_SCALE;
    public static final int ADDRESS_SIZE;

    private static native void registerNatives();

    private Unsafe() {
    }

    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }

    public native int getInt(Object var1, long var2);

    public native void putInt(Object var1, long var2, int var4);

    public native Object getObject(Object var1, long var2);

    public native void putObject(Object var1, long var2, Object var4);

    public native boolean getBoolean(Object var1, long var2);

    public native void putBoolean(Object var1, long var2, boolean var4);

    public native byte getByte(Object var1, long var2);

    public native void putByte(Object var1, long var2, byte var4);

    public native short getShort(Object var1, long var2);

    public native void putShort(Object var1, long var2, short var4);

    public native char getChar(Object var1, long var2);

    public native void putChar(Object var1, long var2, char var4);

    public native long getLong(Object var1, long var2);

    public native void putLong(Object var1, long var2, long var4);

    public native float getFloat(Object var1, long var2);

    public native void putFloat(Object var1, long var2, float var4);

    public native double getDouble(Object var1, long var2);

    public native void putDouble(Object var1, long var2, double var4);

    /** @deprecated */
    @Deprecated
    public int getInt(Object var1, int var2) {
        return this.getInt(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putInt(Object var1, int var2, int var3) {
        this.putInt(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public Object getObject(Object var1, int var2) {
        return this.getObject(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putObject(Object var1, int var2, Object var3) {
        this.putObject(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public boolean getBoolean(Object var1, int var2) {
        return this.getBoolean(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putBoolean(Object var1, int var2, boolean var3) {
        this.putBoolean(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public byte getByte(Object var1, int var2) {
        return this.getByte(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putByte(Object var1, int var2, byte var3) {
        this.putByte(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public short getShort(Object var1, int var2) {
        return this.getShort(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putShort(Object var1, int var2, short var3) {
        this.putShort(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public char getChar(Object var1, int var2) {
        return this.getChar(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putChar(Object var1, int var2, char var3) {
        this.putChar(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public long getLong(Object var1, int var2) {
        return this.getLong(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putLong(Object var1, int var2, long var3) {
        this.putLong(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public float getFloat(Object var1, int var2) {
        return this.getFloat(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putFloat(Object var1, int var2, float var3) {
        this.putFloat(var1, (long)var2, var3);
    }

    /** @deprecated */
    @Deprecated
    public double getDouble(Object var1, int var2) {
        return this.getDouble(var1, (long)var2);
    }

    /** @deprecated */
    @Deprecated
    public void putDouble(Object var1, int var2, double var3) {
        this.putDouble(var1, (long)var2, var3);
    }

    public native byte getByte(long var1);

    public native void putByte(long var1, byte var3);

    public native short getShort(long var1);

    public native void putShort(long var1, short var3);

    public native char getChar(long var1);

    public native void putChar(long var1, char var3);

    public native int getInt(long var1);

    public native void putInt(long var1, int var3);

    public native long getLong(long var1);

    public native void putLong(long var1, long var3);

    public native float getFloat(long var1);

    public native void putFloat(long var1, float var3);

    public native double getDouble(long var1);

    public native void putDouble(long var1, double var3);

    public native long getAddress(long var1);

    public native void putAddress(long var1, long var3);

    public native long allocateMemory(long var1);

    public native long reallocateMemory(long var1, long var3);

    public native void setMemory(Object var1, long var2, long var4, byte var6);

    public void setMemory(long var1, long var3, byte var5) {
        this.setMemory((Object)null, var1, var3, var5);
    }

    public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);

    public void copyMemory(long var1, long var3, long var5) {
        this.copyMemory((Object)null, var1, (Object)null, var3, var5);
    }

    public native void freeMemory(long var1);

    /** @deprecated */
    @Deprecated
    public int fieldOffset(Field var1) {
        return Modifier.isStatic(var1.getModifiers()) ? (int)this.staticFieldOffset(var1) : (int)this.objectFieldOffset(var1);
    }

    /** @deprecated */
    @Deprecated
    public Object staticFieldBase(Class<?> var1) {
        Field[] var2 = var1.getDeclaredFields();

        for(int var3 = 0; var3 < var2.length; ++var3) {
            if (Modifier.isStatic(var2[var3].getModifiers())) {
                return this.staticFieldBase(var2[var3]);
            }
        }

        return null;
    }

    public native long staticFieldOffset(Field var1);

    public native long objectFieldOffset(Field var1);

    public native Object staticFieldBase(Field var1);

    public native boolean shouldBeInitialized(Class<?> var1);

    public native void ensureClassInitialized(Class<?> var1);

    public native int arrayBaseOffset(Class<?> var1);

    public native int arrayIndexScale(Class<?> var1);

    public native int addressSize();

    public native int pageSize();

    public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);

    public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);

    public native Object allocateInstance(Class<?> var1) throws InstantiationException;

    /** @deprecated */
    @Deprecated
    public native void monitorEnter(Object var1);

    /** @deprecated */
    @Deprecated
    public native void monitorExit(Object var1);

    /** @deprecated */
    @Deprecated
    public native boolean tryMonitorEnter(Object var1);

    public native void throwException(Throwable var1);


	//第一个参数var1,表示要操作的对象，第二个参var2表示要操作对象中属性地址的偏移量，第三个参数var4表示需要修改数据的期望的值，第四个参数var5和var6表示需要修改为的新值
    public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);



    public native Object getObjectVolatile(Object var1, long var2);

    public native void putObjectVolatile(Object var1, long var2, Object var4);
	//获取相对于对象var1内存地址的字节偏移量的值，可以认为获取某个内存地址上存放的值
    public native int getIntVolatile(Object var1, long var2);

    public native void putIntVolatile(Object var1, long var2, int var4);

    public native boolean getBooleanVolatile(Object var1, long var2);

    public native void putBooleanVolatile(Object var1, long var2, boolean var4);

    public native byte getByteVolatile(Object var1, long var2);

    public native void putByteVolatile(Object var1, long var2, byte var4);

    public native short getShortVolatile(Object var1, long var2);

    public native void putShortVolatile(Object var1, long var2, short var4);

    public native char getCharVolatile(Object var1, long var2);

    public native void putCharVolatile(Object var1, long var2, char var4);

    public native long getLongVolatile(Object var1, long var2);

    public native void putLongVolatile(Object var1, long var2, long var4);

    public native float getFloatVolatile(Object var1, long var2);

    public native void putFloatVolatile(Object var1, long var2, float var4);

    public native double getDoubleVolatile(Object var1, long var2);

    public native void putDoubleVolatile(Object var1, long var2, double var4);

    public native void putOrderedObject(Object var1, long var2, Object var4);

    public native void putOrderedInt(Object var1, long var2, int var4);

    public native void putOrderedLong(Object var1, long var2, long var4);

    public native void unpark(Object var1);

    public native void park(boolean var1, long var2);

    public native int getLoadAverage(double[] var1, int var2);

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
			//获取原始值
            var5 = this.getIntVolatile(var1, var2);
			//对原始值进行cas加var4
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }

    public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

        return var6;
    }

    public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }

    public final long getAndSetLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var4));

        return var6;
    }

    public final Object getAndSetObject(Object var1, long var2, Object var4) {
        Object var5;
        do {
            var5 = this.getObjectVolatile(var1, var2);
        } while(!this.compareAndSwapObject(var1, var2, var5, var4));

        return var5;
    }

    public native void loadFence();

    public native void storeFence();

    public native void fullFence();

    private static void throwIllegalAccessError() {
        throw new IllegalAccessError();
    }

    static {
        registerNatives();
        Reflection.registerMethodsToFilter(Unsafe.class, new String[]{"getUnsafe"});
        theUnsafe = new Unsafe();
        ARRAY_BOOLEAN_BASE_OFFSET = theUnsafe.arrayBaseOffset(boolean[].class);
        ARRAY_BYTE_BASE_OFFSET = theUnsafe.arrayBaseOffset(byte[].class);
        ARRAY_SHORT_BASE_OFFSET = theUnsafe.arrayBaseOffset(short[].class);
        ARRAY_CHAR_BASE_OFFSET = theUnsafe.arrayBaseOffset(char[].class);
        ARRAY_INT_BASE_OFFSET = theUnsafe.arrayBaseOffset(int[].class);
        ARRAY_LONG_BASE_OFFSET = theUnsafe.arrayBaseOffset(long[].class);
        ARRAY_FLOAT_BASE_OFFSET = theUnsafe.arrayBaseOffset(float[].class);
        ARRAY_DOUBLE_BASE_OFFSET = theUnsafe.arrayBaseOffset(double[].class);
        ARRAY_OBJECT_BASE_OFFSET = theUnsafe.arrayBaseOffset(Object[].class);
        ARRAY_BOOLEAN_INDEX_SCALE = theUnsafe.arrayIndexScale(boolean[].class);
        ARRAY_BYTE_INDEX_SCALE = theUnsafe.arrayIndexScale(byte[].class);
        ARRAY_SHORT_INDEX_SCALE = theUnsafe.arrayIndexScale(short[].class);
        ARRAY_CHAR_INDEX_SCALE = theUnsafe.arrayIndexScale(char[].class);
        ARRAY_INT_INDEX_SCALE = theUnsafe.arrayIndexScale(int[].class);
        ARRAY_LONG_INDEX_SCALE = theUnsafe.arrayIndexScale(long[].class);
        ARRAY_FLOAT_INDEX_SCALE = theUnsafe.arrayIndexScale(float[].class);
        ARRAY_DOUBLE_INDEX_SCALE = theUnsafe.arrayIndexScale(double[].class);
        ARRAY_OBJECT_INDEX_SCALE = theUnsafe.arrayIndexScale(Object[].class);
        ADDRESS_SIZE = theUnsafe.addressSize();
    }
}

```

JUC包中大部分功能都是依靠CAS操作完成的

### 原子更新基本类型

原子更新基本类型主要包括AtomicInteger，AtomicLong,AtomicBoolean，这三个类提供的方法基本一致，以AtomicInteger为例查看该类的方法

```java
package java.util.concurrent.atomic;
import java.util.function.IntUnaryOperator;
import java.util.function.IntBinaryOperator;
import sun.misc.Unsafe;


public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
			//可以认为是获取 AtomicInteger对象value字段相对于对象的起始内存地址的字节偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    //返回当前值
    public final int get() {
        return value;
    }

    //设置当前值
    public final void set(int newValue) {
        value = newValue;
    }

    //最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    //原子的方式设置为newValue并返回旧值
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    //如果当前值等于预期值expect，则以原子方式将该值设置为输入的值。
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * <p><a href="package-summary.html#weakCompareAndSet">May fail
     * spuriously and does not provide ordering guarantees</a>, so is
     * only rarely an appropriate alternative to {@code compareAndSet}.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful
     */
    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    //以原子方式将当前值加1并返回旧值
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    //以原子方式将当前值减1并返回旧值
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }

    //以原子方式将当前值加delta并返回旧值
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    //以原子方式将当前值加1并返回新值
    public final int incrementAndGet() {
		//获取指定内存地址的值并加1
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

      //以原子方式将当前值减1并返回新值
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }

    //以原子方式将当前值加delta并返回新值
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }

    //将当前值设置为IntUnaryOperator的applyAsInt方法返回的值，并返回当前值
    public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }

    //将当前值设置为IntUnaryOperator的applyAsInt方法返回的值，并返回新值
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

测试代码

```java
   @Test
    public void testGetAndAdd()throws Exception{
        AtomicInteger count = new AtomicInteger(0);

        int andAdd = count.getAndAdd(10);
        //返回之前的值10
        System.out.println(andAdd);
        int i = count.addAndGet(10);
        //返回新值20，因为前面加了两次10
        System.out.println(i);

        int getAndUpdate=count.getAndUpdate(new IntUnaryOperator() {
            @Override
            public int applyAsInt(int operand) {
                return operand+10;
            }
        });
        System.out.println(getAndUpdate);
        //返回30,因为已经加了三次10
        System.out.println(count.get());

        int i1 = count.incrementAndGet();
        //输出31
        System.out.println(i1);


    }
```

### 原子更新数组类型

原子更新数组类型包括AtomicIntegerArray，AtomicLongArray和AtomicReferenceArray，这三个类提供的方法基本一致，以AtomicIntegerArray为例查看该类的方法

```java
package java.util.concurrent.atomic;
import java.util.function.IntUnaryOperator;
import java.util.function.IntBinaryOperator;
import sun.misc.Unsafe;


public class AtomicIntegerArray implements java.io.Serializable {
    private static final long serialVersionUID = 2862133569453604235L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;
    private final int[] array;

    static {
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }
	//检查是否索引越界
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);

        return byteOffset(i);
    }

    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }

    
    public AtomicIntegerArray(int length) {
        array = new int[length];
    }

    
    public AtomicIntegerArray(int[] array) {
        // Visibility guaranteed by final field guarantees
        this.array = array.clone();
    }

    //返回数组长度
    public final int length() {
        return array.length;
    }

    //返回数组索引i的当前的值
    public final int get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    private int getRaw(long offset) {
        return unsafe.getIntVolatile(array, offset);
    }

    //将索引下标为i所对应的值设置为newValue
    public final void set(int i, int newValue) {
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
    }

  
    public final void lazySet(int i, int newValue) {
        unsafe.putOrderedInt(array, checkedByteOffset(i), newValue);
    }

    /**
     * Atomically sets the element at position {@code i} to the given
     * value and returns the old value.
     *
     * @param i the index
     * @param newValue the new value
     * @return the previous value
     */
    public final int getAndSet(int i, int newValue) {
        return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
    }

    //如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update
    public final boolean compareAndSet(int i, int expect, int update) {
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    private boolean compareAndSetRaw(long offset, int expect, int update) {
        return unsafe.compareAndSwapInt(array, offset, expect, update);
    }

    /**
     * Atomically sets the element at position {@code i} to the given
     * updated value if the current value {@code ==} the expected value.
     *
     * <p><a href="package-summary.html#weakCompareAndSet">May fail
     * spuriously and does not provide ordering guarantees</a>, so is
     * only rarely an appropriate alternative to {@code compareAndSet}.
     *
     * @param i the index
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful
     */
    public final boolean weakCompareAndSet(int i, int expect, int update) {
        return compareAndSet(i, expect, update);
    }

    /**
     * Atomically increments by one the element at index {@code i}.
     *
     * @param i the index
     * @return the previous value
     */
    public final int getAndIncrement(int i) {
        return getAndAdd(i, 1);
    }

    /**
     * Atomically decrements by one the element at index {@code i}.
     *
     * @param i the index
     * @return the previous value
     */
    public final int getAndDecrement(int i) {
        return getAndAdd(i, -1);
    }

    /**
     * Atomically adds the given value to the element at index {@code i}.
     *
     * @param i the index
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int i, int delta) {
        return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
    }

    /**
     * Atomically increments by one the element at index {@code i}.
     *
     * @param i the index
     * @return the updated value
     */
    public final int incrementAndGet(int i) {
        return getAndAdd(i, 1) + 1;
    }

    /**
     * Atomically decrements by one the element at index {@code i}.
     *
     * @param i the index
     * @return the updated value
     */
    public final int decrementAndGet(int i) {
        return getAndAdd(i, -1) - 1;
    }

    //以原子方式将输入值与数组中索引i的元素相加
    public final int addAndGet(int i, int delta) {
        return getAndAdd(i, delta) + delta;
    }


    /**
     * Atomically updates the element at index {@code i} with the results
     * of applying the given function, returning the previous value. The
     * function should be side-effect-free, since it may be re-applied
     * when attempted updates fail due to contention among threads.
     *
     * @param i the index
     * @param updateFunction a side-effect-free function
     * @return the previous value
     * @since 1.8
     */
    public final int getAndUpdate(int i, IntUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSetRaw(offset, prev, next));
        return prev;
    }

    /**
     * Atomically updates the element at index {@code i} with the results
     * of applying the given function, returning the updated value. The
     * function should be side-effect-free, since it may be re-applied
     * when attempted updates fail due to contention among threads.
     *
     * @param i the index
     * @param updateFunction a side-effect-free function
     * @return the updated value
     * @since 1.8
     */
    public final int updateAndGet(int i, IntUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }

    /**
     * Atomically updates the element at index {@code i} with the
     * results of applying the given function to the current and
     * given values, returning the previous value. The function should
     * be side-effect-free, since it may be re-applied when attempted
     * updates fail due to contention among threads.  The function is
     * applied with the current value at index {@code i} as its first
     * argument, and the given update as the second argument.
     *
     * @param i the index
     * @param x the update value
     * @param accumulatorFunction a side-effect-free function of two arguments
     * @return the previous value
     * @since 1.8
     */
    public final int getAndAccumulate(int i, int x,
                                      IntBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSetRaw(offset, prev, next));
        return prev;
    }

    /**
     * Atomically updates the element at index {@code i} with the
     * results of applying the given function to the current and
     * given values, returning the updated value. The function should
     * be side-effect-free, since it may be re-applied when attempted
     * updates fail due to contention among threads.  The function is
     * applied with the current value at index {@code i} as its first
     * argument, and the given update as the second argument.
     *
     * @param i the index
     * @param x the update value
     * @param accumulatorFunction a side-effect-free function of two arguments
     * @return the updated value
     * @since 1.8
     */
    public final int accumulateAndGet(int i, int x,
                                      IntBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }

    /**
     * Returns the String representation of the current values of array.
     * @return the String representation of the current values of array
     */
    public String toString() {
        int iMax = array.length - 1;
        if (iMax == -1)
            return "[]";

        StringBuilder b = new StringBuilder();
        b.append('[');
        for (int i = 0; ; i++) {
            b.append(getRaw(byteOffset(i)));
            if (i == iMax)
                return b.append(']').toString();
            b.append(',').append(' ');
        }
    }

}
```

测试例子

```java
    @Test
    public void test1() {
        int[] array = {0, 1, 2, 3, 4, 5, 6};
        AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(array);
        //返回索引3对应的值
        int i = atomicIntegerArray.get(3);
        System.out.println(i);

        atomicIntegerArray.set(3, 4);
        int ii = atomicIntegerArray.get(3);
        System.out.println(ii);
        //预期值为4，并设置新值为5，此时再获取数组下标为3的值就为5
        boolean b = atomicIntegerArray.compareAndSet(3, 4, 5);
        System.out.println(b);
        int iii = atomicIntegerArray.get(3);
        System.out.println(iii);
    }
```

### 原子更新引用类型

原子更新数组类型包括如下三类

AtomicReference原子更新引用类型

AtomicReferenceFieldUpdater：原子更新引用类型里的字段

AtomicMarkableReference：原子更新带有标记位的引用类型

里面的方法比较简单，我们直接看示例

```java
public class AtomicReferenceTest {

    @Test
    public void test1()throws Exception{
        AtomicReference<Student>  atomicReference=new AtomicReference<>();

        Student student=new Student();
        student.setAge(23);
        student.setName("张三");

        Student newStudent=new Student();
        newStudent.setName("李四");
        newStudent.setAge(24);

        atomicReference.set(student);
        System.out.println(atomicReference.get());

        atomicReference.compareAndSet(student,newStudent);
        System.out.println(atomicReference.get());




    }

    static class Student{
        private String name;
        private int age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "Student{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}

```

```java
public class AtomicMarkableReferenceTest {
    @Test
    public void test1() throws Exception {

        Student student = new Student();
        student.setAge(23);
        student.setName("张三");

        Student newStudent = new Student();
        newStudent.setName("李四");
        newStudent.setAge(24);

        AtomicMarkableReference<Student> atomicReference = new AtomicMarkableReference<>(student,false);
        //初始化的值
        System.out.println(atomicReference.isMarked());
        System.out.println(atomicReference.getReference());

        //修改失败
        boolean b = atomicReference.compareAndSet(student, newStudent, true, false);
        System.out.println(atomicReference.isMarked());
        System.out.println(b);
        System.out.println(atomicReference.getReference());


        //修改成功
        boolean b1 = atomicReference.compareAndSet(student, newStudent, false, true);
        System.out.println(atomicReference.isMarked());
        System.out.println(b1);
        System.out.println(atomicReference.getReference());



    }

    static class Student {
        private String name;
        private int age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "Student{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}

```

### 原子更新字段类

AtomicIntegerFieldUpdater：原子更新整型的字段的更新器
AtomicLongFieldUpdater：原子更新长整型字段的更新器
AtomicStampedReference：原子更新带有版本号的引用类型,该类可以解决更新时的ABA问题

以上三个类的方法几乎一致，也比较简单，我们直接看一下下面这个测试实例

```java
public class AtomicIntegerFieldUpdaterTest {

    @Test
    public void test1()throws Exception{
        AtomicIntegerFieldUpdater<Student> atomicIntegerFieldUpdater= AtomicIntegerFieldUpdater.newUpdater(Student.class,"age");
        Student student=new Student();
        student.setAge(23);
        student.setName("张三");
        System.out.println(atomicIntegerFieldUpdater.get(student));
        //直接返老还童
        boolean b = atomicIntegerFieldUpdater.compareAndSet(student, 23, 2);
        System.out.println(b);
        System.out.println(atomicIntegerFieldUpdater.get(student));
    }




    static class Student{
        public String name;
        //必须为public,且用volatile修饰，否则会报错
        public volatile int age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "Student{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}

```

