ByteBuffer

### ByteBuffer字节缓冲区源码

```java
package java.nio;

//字节缓冲区，继承Buffer
public abstract class ByteBuffer
    extends Buffer
    implements Comparable<ByteBuffer>
{

	//仅用于堆缓冲区，用来存储字节的数组
    final byte[] hb;                  // Non-null only for heap buffers
	//hb数组的偏移量
    final int offset;
	//是否为只读，仅对堆缓冲区有效
    boolean isReadOnly;                 // Valid only for heap buffers

	//根据给定的标记、位置、限制、容量、byte数组和数组偏移量创建一个新的缓冲区
    ByteBuffer(int mark, int pos, int lim, int cap,   // package-private
                 byte[] hb, int offset)
    {
        super(mark, pos, lim, cap);
        this.hb = hb;
        this.offset = offset;
    }

  
    ByteBuffer(int mark, int pos, int lim, int cap) { // package-private
        this(mark, pos, lim, cap, null, 0);
    }



    //分配一个新的直接字节缓冲区
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }



    //分配一个新的堆字节缓冲区
    public static ByteBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapByteBuffer(capacity, capacity);
    }

    //将字节数组包装到缓冲区中
    public static ByteBuffer wrap(byte[] array,
                                    int offset, int length)
    {
        try {
            return new HeapByteBuffer(array, offset, length);
        } catch (IllegalArgumentException x) {
            throw new IndexOutOfBoundsException();
        }
    }

    //将字节数组包装到缓冲区中
    public static ByteBuffer wrap(byte[] array) {
        return wrap(array, 0, array.length);
    }


    //创建一个新的字节缓冲区，其内容是这个缓冲区内容的共享子序列
	//新缓冲区的位置将为零，其容量和限制将为该缓冲区中剩余的字节数，其标记将未定义
    public abstract ByteBuffer slice();

    //创建一个共享该缓冲区内容的新字节缓冲区
	//新缓冲区的容量、限制、位置和标记值与此缓冲区的相同
    public abstract ByteBuffer duplicate();

    //创建一个共享该缓冲区内容的新的只读字节缓冲区
	//新缓冲区的容量、限制、位置和标记值与此缓冲区的相同
    public abstract ByteBuffer asReadOnlyBuffer();

    // -- Singleton get/put methods --

    //读取该缓冲区当前位置的字节,然后对位置进行增加
    public abstract byte get();

    //将给定的字节按当前位置写入该缓冲区，然后对位置进行递增
    public abstract ByteBuffer put(byte b);

    //绝对的get方法，读取给定索引处的字节
    public abstract byte get(int index);

    //绝对的put方法，将给定字节写入此缓冲区的给定索引处
    public abstract ByteBuffer put(int index, byte b);

    // -- Bulk get operations --

	 //相对批量获取方法，将缓冲区中的字节从offset位置，取length长度复制到dst字节数组中
    public ByteBuffer get(byte[] dst, int offset, int length) {
        checkBounds(offset, length, dst.length);
        if (length > remaining())
            throw new BufferUnderflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            dst[i] = get();
        return this;
    }

    //相对批量获取方法，将缓冲区中的字节从0位置，取dst.length长度复制到dst字节数组中
    public ByteBuffer get(byte[] dst) {
        return get(dst, 0, dst.length);
    }


    // -- Bulk put operations --

    //相对批量put方法，将src缓冲区的字节数组追加到当前字节缓冲区中
    public ByteBuffer put(ByteBuffer src) {
        if (src == this)
            throw new IllegalArgumentException();
        if (isReadOnly())
            throw new ReadOnlyBufferException();
        int n = src.remaining();
        if (n > remaining())
            throw new BufferOverflowException();
        for (int i = 0; i < n; i++)
            put(src.get());
        return this;
    }

    //相对批量put方法,将src字节数组中的从offset位置开始，取length长度追加到当前字节缓冲区中
    public ByteBuffer put(byte[] src, int offset, int length) {
        checkBounds(offset, length, src.length);
        if (length > remaining())
            throw new BufferOverflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            this.put(src[i]);
        return this;
    }

    ////相对批量put方法,将src字节数组追加到当前字节缓冲区中
    public final ByteBuffer put(byte[] src) {
        return put(src, 0, src.length);
    }

    // -- Other stuff --

    //此缓冲区是否由可访问的字节数组支持
    public final boolean hasArray() {
        return (hb != null) && !isReadOnly;
    }

    //返回返回此缓冲区的字节数组
    public final byte[] array() {
        if (hb == null)
            throw new UnsupportedOperationException();
        if (isReadOnly)
            throw new ReadOnlyBufferException();
        return hb;
    }

    //返回缓冲区的第一个元素在该缓冲区的后备数组中的偏移量
    public final int arrayOffset() {
        if (hb == null)
            throw new UnsupportedOperationException();
        if (isReadOnly)
            throw new ReadOnlyBufferException();
        return offset;
    }

    //压缩此缓冲区
    public abstract ByteBuffer compact();

    //字节缓冲区是否直接缓冲区
    public abstract boolean isDirect();


    public String toString() {
        StringBuffer sb = new StringBuffer();
        sb.append(getClass().getName());
        sb.append("[pos=");
        sb.append(position());
        sb.append(" lim=");
        sb.append(limit());
        sb.append(" cap=");
        sb.append(capacity());
        sb.append("]");
        return sb.toString();
    }

    public int hashCode() {
        int h = 1;
        int p = position();
        for (int i = limit() - 1; i >= p; i--)



            h = 31 * h + (int)get(i);

        return h;
    }

    public boolean equals(Object ob) {
        if (this == ob)
            return true;
        if (!(ob instanceof ByteBuffer))
            return false;
        ByteBuffer that = (ByteBuffer)ob;
        if (this.remaining() != that.remaining())
            return false;
        int p = this.position();
        for (int i = this.limit() - 1, j = that.limit() - 1; i >= p; i--, j--)
            if (!equals(this.get(i), that.get(j)))
                return false;
        return true;
    }

    private static boolean equals(byte x, byte y) {



        return x == y;

    }

    //将这个缓冲区与另一个缓冲区进行比较
    public int compareTo(ByteBuffer that) {
        int n = this.position() + Math.min(this.remaining(), that.remaining());
        for (int i = this.position(), j = that.position(); i < n; i++, j++) {
            int cmp = compare(this.get(i), that.get(j));
            if (cmp != 0)
                return cmp;
        }
        return this.remaining() - that.remaining();
    }

    private static int compare(byte x, byte y) {






        return Byte.compare(x, y);

    }

	//默认为大端字节
    boolean bigEndian                                   // package-private
        = true;
    boolean nativeByteOrder                             // package-private
        = (Bits.byteOrder() == ByteOrder.BIG_ENDIAN);

    //缓冲区的字节顺序
    public final ByteOrder order() {
        return bigEndian ? ByteOrder.BIG_ENDIAN : ByteOrder.LITTLE_ENDIAN;
    }

    //修改这个缓冲区的字节顺序
    public final ByteBuffer order(ByteOrder bo) {
        bigEndian = (bo == ByteOrder.BIG_ENDIAN);
        nativeByteOrder =
            (bigEndian == (Bits.byteOrder() == ByteOrder.BIG_ENDIAN));
        return this;
    }

    // Unchecked accessors, for use by ByteBufferAs-X-Buffer classes
    //
    abstract byte _get(int i);                          // package-private
    abstract void _put(int i, byte b);                  // package-private


    //读取字符值的相对get方法
    public abstract char getChar();

    //用于写入字符值的相对put方法
    public abstract ByteBuffer putChar(char value);

    //读取字符值的绝对get方法
    public abstract char getChar(int index);

    //用于写入字符值的绝对put方法
    public abstract ByteBuffer putChar(int index, char value);

    //创建一个字节缓冲区作为char缓冲区的视图
    public abstract CharBuffer asCharBuffer();

    //获取short的相对get方法
    public abstract short getShort();

    public abstract ByteBuffer putShort(short value);

    public abstract short getShort(int index);
 
     * Absolute <i>put</i> method for writing a short
     * value&nbsp;&nbsp;<i>(optional operation)</i>.
     *
     * <p> Writes two bytes containing the given short value, in the
     * current byte order, into this buffer at the given index.  </p>
     *
     * @param  index
     *         The index at which the bytes will be written
     *
     * @param  value
     *         The short value to be written
     *
     * @return  This buffer
     *
     * @throws  IndexOutOfBoundsException
     *          If <tt>index</tt> is negative
     *          or not smaller than the buffer's limit,
     *          minus one
     *
     * @throws  ReadOnlyBufferException
     *          If this buffer is read-only
     */
    public abstract ByteBuffer putShort(int index, short value);
	//创建一个字节缓冲区作为short缓冲区的视图
    public abstract ShortBuffer asShortBuffer();

    public abstract int getInt();

    public abstract ByteBuffer putInt(int value);

    public abstract int getInt(int index);

    public abstract ByteBuffer putInt(int index, int value);
	//创建一个字节缓冲区作为int缓冲区的视图
    public abstract IntBuffer asIntBuffer();

    public abstract long getLong();

    public abstract ByteBuffer putLong(long value);

    public abstract long getLong(int index);

    public abstract ByteBuffer putLong(int index, long value);

    public abstract LongBuffer asLongBuffer();

    public abstract float getFloat();

    public abstract ByteBuffer putFloat(float value);

    public abstract float getFloat(int index);

    public abstract ByteBuffer putFloat(int index, float value);

    public abstract FloatBuffer asFloatBuffer();

    public abstract double getDouble();

    public abstract ByteBuffer putDouble(double value);

    public abstract double getDouble(int index);
	
    public abstract ByteBuffer putDouble(int index, double value);

    public abstract DoubleBuffer asDoubleBuffer();

}

```

### 字节顺序

基本数据类型及其大小

| 数据类型 | 大小(以字节表示) |
| -------- | ---------------- |
| Byte     | 1                |
| Char     | 2                |
| Short    | 2                |
| Int      | 4                |
| Long     | 8                |
| Float    | 4                |
| Double   | 8                |

字节顺序：
指多字节数据在计算机内存中存储或者网络传输时各字节的存储顺序，有大端和小端两种方式
大端：
指高位字节存放在内存的低地址端，低位字节存放在内存的高地址端。
小端：
指低位字节放在内存的低地址端，高位字节放在内存的高地址端。

以int类型的数据 0x037fb4c7，十进制的为199+180*2^8+127*2^16+3*2^24=199+180*256+127*65536+3*16777216=58700999。

大端字节顺序-BIG_ENDIAN

![1645424784](.\pic\1645424784.png)

小端字节顺序-LITTLE_ENDIAN

![1645424810](.\pic\1645424810.png)

字节顺序通常是由硬件决定的，我们可以通过如下代码查找我们电脑系统的本地字节顺序。网络通信中也会涉及到大端字节和小端字节，需要统一规范，所以，IP协议规定了使用大端的网络字节顺序

```java
    @Test
    public void nativeOrder()throws Exception{

        System.out.println(ByteOrder.nativeOrder());
    }
}
```

ByteBuffer默认字节顺序总是ByteBuffer.BIG_ENDIAN，无论系统的固有字节顺序是什么。Java的默认字节顺序是大端字节顺序，这允许类文件等以及串行化的对象可以在任何JVM中工作。如果固有硬件字节顺序是小端，这会有性能隐患。在使用固有硬件字节顺序时，将ByteBuffer的内容当作其他数据类型存取（很快就会讨论到）很可能高效得多。

如图：字节转换为字符视图，So对应的byte十进制为83和111，由于char类型占用两个字节，按照大端字节转换则为83*2^8+111=873*256+111=21356,对应的汉子字符刚好是卯

![1645426492](.\pic\1645426492.png)

### DirectByteBuffer直接缓冲区

直接缓冲区时I/O的最佳选择，但可能比创建非直接缓冲区要花费更高的成本。直接缓冲区使用的内存是通过调用本地操作系统方面的代码分配的，绕过了标准JVM堆栈。建立和销毁直接缓冲区会明显比具有堆栈的缓冲区更加破费，这取决于主操作系统以及JVM实现。直接缓冲区的内存区域不受无用存储单元收集支配，因为它们位于标准JVM堆栈之外。

直接ByteBuffer是通过调用具有所需容量的ByteBuffer.allocateDirect()函数产生的。

如下代码使用的是直接缓冲区

```java

    @Test
    public void TesDirectByteBuffer1(){

        //填充
        ByteBuffer bf = ByteBuffer.allocateDirect(10);
        bf.put((byte) 'M');
        bf.put((byte) 'e');
        bf.put((byte) 'l');
        bf.put((byte) 'l');
        bf.put((byte) 'o');
        bf.put((byte) 'w');

        //进行翻转
        bf.flip();
        //释放
        for (int i = 0; bf.hasRemaining(); i++) {
            byte b1 = bf.get();
            //  System.out.println(b1);
//            //打印出16进制
//            String hex = Integer.toHexString(b1 & 0xFF);
//            System.out.println(hex);
            System.out.println((char) b1);
        }



    }
```

### HeapByteBuffer堆缓冲区

是用于可读写的堆缓冲区，操作在java的堆中进行，实际上就是操作系统中的用户内存，而不是内核内存。性能相对直接缓冲区稍微差一些。