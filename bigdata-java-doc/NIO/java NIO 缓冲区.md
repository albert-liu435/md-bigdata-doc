java NIO 缓冲区

缓冲区(Buffer)是中NIO中基础的内容,存在于包java.nio下面。缓冲区本质上是一个可以读写数据的内存块，可以理解成是一个容器对象，该对象内部含有一个存储数据的数组和一组方法，可以更轻松地使用内存块，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 Buffer。

### 缓冲区操作

缓冲区，以及缓冲区如何工作，是所有 I/O 的基础。所谓“输入／输出”讲的无非就是把数据移进或移出缓冲区。

进程执行 I/O 操作，归结起来，也就是向操作系统发出请求，让它要么把缓冲区里的数据排干（写），要么用数据把缓冲区填满（读）

简单用如下图展示

![1645062990](.\pic\1645062990.png)

用户空间的进程可以认为是java程序，而内核空间可以认为就是操作系统。

### Buffer的继承关系

![Buffer](.\pic\Buffer.png)

- Buffer:是所有buffer的顶级父类，里面主要定义了一些公共属性和方法
  - ByteBuffer:定义了用来操作byte字节的缓冲区
    - HeapByteBuffer：是用于可读写的堆缓冲区，操作在java的堆中进行，HeapByteBufferR是只用于读的堆缓冲区
    - DirectByteBuffer:是用于读写的直接缓冲区，操作在java的堆外执行即直接操作物理内存，是比HeapByteBuffer操作更高效的缓冲区
  - CharBuffer:字符缓冲区，里面存放的是Char数组，用来进行字符操作存取。
  - IntBuffer:Int类型缓冲区，里面存放的是int数组，用来进行int数值型操作
  - 同理FloatBuffer,DoubleBuffer,ShortBuffer,LongBuffer

### Buffer源码

Buffer的源码如下，里面定义了一些公共的属性和方法

```java

package java.nio;

import java.util.Spliterator;

//用于特定原始类型的数据的容器，是NIO缓冲区的基类，
public abstract class Buffer {

    
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: 0<mark <= position <= limit <= capacity
	//一个备忘位置。调用mark( )来设定mark = postion。调用reset( )设定position =mark
    private int mark = -1;
	//下一个要被读或写的元素的索引。位置会自动由相应的get( )和put( )函数更新。
    private int position = 0;
	//缓冲区的第一个不能被读或写的元素。或者说，缓冲区中现存元素的计数
    private int limit;
	//缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定，并且永远不能被改变
    private int capacity;

    // 仅用于直接缓冲区
    long address;

    // 根据给定的标记、位置、限制和容量来创建一个缓冲区
    Buffer(int mark, int pos, int lim, int cap) {       // package-private
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }

    //缓冲区容量
    public final int capacity() {
        return capacity;
    }

    //缓冲区的位置
    public final int position() {
        return position;
    }

    //设置缓冲区的位置,如果标记已定义且大于新位置，则丢弃该标记
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }

    //缓冲区的限制
    public final int limit() {
        return limit;
    }

    //设置缓冲区的限制，如果位置大于新的限制，那么它被设置为新的限制，如果标记被定义并且大于新的限制，那么它将被丢弃。

    public final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }

    //将缓冲区的标记设置在它的位置上。
    public final Buffer mark() {
        mark = position;
        return this;
    }

    //将此缓冲区的位置重置为以前标记的位置。
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }

    //清除这个缓冲区，位置被设置为零，限制被设置为容量，并且标记被丢弃。
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

    //翻转这个缓冲区。限制被设置为当前位置，然后位置被设置为零。
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }

    //重绕缓冲区
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }

    //返回当为位置与上界之间的元素数
    public final int remaining() {
        return limit - position;
    }

    //缓存的位置与上界之间是否还有元素
    public final boolean hasRemaining() {
        return position < limit;
    }

    //缓冲区是否为只读缓冲区
    public abstract boolean isReadOnly();

    //告诉这个缓冲区是否由可访问的数组
    public abstract boolean hasArray();

    //返回支持此缓冲区的数组
    public abstract Object array();

    //返回该缓冲区的缓冲区的第一个元素的背衬数组中的偏移量
    public abstract int arrayOffset();

    //告诉这个缓冲区是否是 direct
    public abstract boolean isDirect();

	 //获取元素时，根据limit检查当前位置，如果不小于limit，则抛出异常，否则对position加1
    final int nextGetIndex() {                          // package-private
        if (position >= limit)
            throw new BufferUnderflowException();
        return position++;
    }
	
	//
    final int nextGetIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferUnderflowException();
        int p = position;
        position += nb;
        return p;
    }

    //添加元素时，根据limit检查当前位置，如果不小于limit，则抛出异常，否则对position加1
    final int nextPutIndex() {                          // package-private
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }

    final int nextPutIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }

    //根据给定的索引查询是否分号限制
    final int checkIndex(int i) {                       // package-private
        if ((i < 0) || (i >= limit))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int checkIndex(int i, int nb) {               // package-private
        if ((i < 0) || (nb > limit - i))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int markValue() {                             // package-private
        return mark;
    }

	//可以跟数据库的truncate进行比较
    final void truncate() {                             // package-private
        mark = -1;
        position = 0;
        limit = 0;
        capacity = 0;
    }

    final void discardMark() {                          // package-private
        mark = -1;
    }

    static void checkBounds(int off, int len, int size) { // package-private
        if ((off | len | (off + len) | (size - (off + len))) < 0)
            throw new IndexOutOfBoundsException();
    }

}

```

### Buffer缓冲区的存取

缓冲区的存取需要经过填充->翻转->释放。一旦缓冲区对象完成填充并释放，它就可以被重新使用了。

下面以BtyeBuffer为例展示缓冲区的填充翻转过程

```java
    @Test
    public void TestHeapByteBuffer1() throws Exception {
        //填充
        ByteBuffer bf = ByteBuffer.allocate(10);
        bf.put((byte) 'M');
        bf.put((byte) 'e');
        bf.put((byte) 'l');
        bf.put((byte) 'l');
        bf.put((byte) 'w');

        //进行翻转
        bf.flip();

        //释放
        for (int i = 0; bf.hasRemaining(); i++) {
            byte b1 = bf.get();
            //  System.out.println(b1);
            //打印出16进制
            String hex = Integer.toHexString(b1 & 0xFF);
            System.out.println(hex);
        }


    }
```

缓冲区填充示意图

![1645150378](.\pic\1645150378.png)

缓冲区翻转示意图

![1645150429](.\pic\1645150429.png)

### Buffer缓冲区的压缩

如果从缓冲区中释放一部分数据，而不是全部，然后重新填充。为了实现这一点，未读的数据元素需要下移以使第一个元素索引为0。

缓冲区压缩实例代码

```java
    @Test
    public void TestHeapByteBuffer2()throws Exception{
        //填充
        ByteBuffer bf = ByteBuffer.allocate(10);
        bf.put((byte) 'M');
        bf.put((byte) 'e');
        bf.put((byte) 'l');
        bf.put((byte) 'l');
        bf.put((byte) 'o');
        bf.put((byte) 'w');

        //进行翻转
        bf.flip();

        //释放两个字节元素的缓冲区
        for (int i = 0; i<2; i++) {
            byte b1 = bf.get();
            //  System.out.println(b1);
            //打印出16进制
            String hex = Integer.toHexString(b1 & 0xFF);
            System.out.println(hex);
        }

        System.out.println("启用压缩");
        bf.compact();
        //进行翻转
        bf.flip();

        //释放
        for (int i = 0; bf.hasRemaining(); i++) {
            byte b1 = bf.get();
            //  System.out.println(b1);
            //打印出16进制
            String hex = Integer.toHexString(b1 & 0xFF);
            System.out.println(hex);
        }



    }
```

被部分释放的缓冲区

![1645151522](.\pic\1645151522.png)

压缩后的buffer![1645151558](.\pic\1645151558.png)