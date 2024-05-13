字节数组流ByteArrayInputStream&ByteArrayOutputStream

### ByteArrayInputStream

ByteArrayInputStream 是字节数组输入流。它继承于InputStream。它包含一个内部缓冲区，该缓冲区包含从流中读取的字节；ByteArrayInputStream本质就是通过字节数组来实现的。同时ByteArrayInputStream 的内部额外的定义了一个计数器，它被用来跟踪 read() 方法要读取的下一个字节。

```java

package java.io;

//包含一个字节缓冲区，用于从流中读取字节
public
class ByteArrayInputStream extends InputStream {

    //用于存储从流中读取的字节数据
    protected byte buf[];

    //表示下一个即将读取的字节索引
    protected int pos;

    //流中当前标记的位置,ByteArrayInputStream对象在构造时默认被标记在位置0
    protected int mark = 0;

    //字节流的总长度
    protected int count;

    //用buf作为它的缓冲区数组创建ByteArrayInputStream对象
    public ByteArrayInputStream(byte buf[]) {
        this.buf = buf;
        this.pos = 0;
        this.count = buf.length;
    }

    //用buf作为它的缓冲区数组创建ByteArrayInputStream对象，pos的初始值为offset，count的初始值是offset+length和buf.length的最小值
    public ByteArrayInputStream(byte buf[], int offset, int length) {
        this.buf = buf;
        this.pos = offset;
        this.count = Math.min(offset + length, buf.length);
        this.mark = offset;
    }

    //从输入流中读取下一个字节的数据
    public synchronized int read() {
        return (pos < count) ? (buf[pos++] & 0xff) : -1;
    }

    //从输入流中读取最多len长度的字节数据到b数组中
    public synchronized int read(byte b[], int off, int len) {
       //字节数组为null,则抛出异常
        if (b == null) {
            throw new NullPointerException();
            //off < 0 || len < 0 || len > b.length - off 的情况下抛出数组越界
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        }
		//表示已经超出了字节缓冲区的大小，所以返回-1
        if (pos >= count) {
            return -1;
        }
		//表示缓冲区可以容纳的字节大小
        int avail = count - pos;
        if (len > avail) {
            len = avail;
        }
        if (len <= 0) {
            return 0;
        }
        //进行复制
        System.arraycopy(buf, pos, b, off, len);
        pos += len;
        return len;
    }

    //跳过n个字节
    public synchronized long skip(long n) {
        long k = count - pos;
        if (n < k) {
            k = n < 0 ? 0 : n;
        }

        pos += k;
        return k;
    }

    //返回可从此输入流读取(或跳过)的剩余字节数
    public synchronized int available() {
        return count - pos;
    }

    //是否支持标记操作
    public boolean markSupported() {
        return true;
    }

    //设置流中的当前标记位置
    public void mark(int readAheadLimit) {
        mark = pos;
    }

    //将缓冲区重置到标记的位置
    public synchronized void reset() {
        pos = mark;
    }

    //关闭ByteArrayInputStream
    public void close() throws IOException {
    }

}

```

简单实例

```java
    @Test
    public void test1(){


        byte[] bytes = { 97, 98, 99, 100, 101, 102,
                103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113,
                114, 115, 116, 117, 118, 119, 120, 121, 122 };
        System.out.println(new String(datas, 0, datas.length));

        ByteArrayInputStream byteArrayInputStream=new ByteArrayInputStream(bytes);
        int len;
        while ((len=byteArrayInputStream.read()) !=-1){
            System.out.println((char) len);
        }
        System.out.println("==================");
        //将pos重置为0
        byteArrayInputStream.reset();
        //将pos向后跳跃5个字节
        byteArrayInputStream.skip(5);
        //将此处做上标记，下次reset时，pos定位在标记处，这里的参数0无实际意义
        byteArrayInputStream.mark(0);
        System.out.println("\r\n"+(char)byteArrayInputStream.read());
        byteArrayInputStream.skip(10);
        System.out.println((char)byteArrayInputStream.read());
        byteArrayInputStream.reset();
        System.out.println((char)byteArrayInputStream.read());


    }
```

### ByteArrayOutputStream

ByteArrayOutputStream 是字节数组输出流。它继承于OutputStream。ByteArrayOutputStream 中的数据被写入一个 byte 数组。缓冲区会随着数据的不断写入而自动增长。可使用 toByteArray() 和 toString() 获取数据。

```java
package java.io;

import java.util.Arrays;

//继承OutputStream，封装了一个byte类型的数组buf作为数据的缓存区，当进行数据写人的时候，数据会不断地写入buf缓存区中，缓冲区的默认大小为32
public class ByteArrayOutputStream extends OutputStream {

    //字节缓冲区，用来存储字节数据
    protected byte buf[];

    //缓冲区中的有效字节数量
    protected int count;

    //创建一个默认大小为32字节的ByteArrayOutputStream
    public ByteArrayOutputStream() {
        this(32);
    }
	//构造方法
    public ByteArrayOutputStream(int size) {
        if (size < 0) {
            throw new IllegalArgumentException("Negative initial size: "
                                               + size);
        }
        buf = new byte[size];
    }

    //确保字节缓冲区的容量足够
    private void ensureCapacity(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - buf.length > 0)
			//进行扩容
            grow(minCapacity);
    }

    //分配数组的最大大小
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //进行扩容
    private void grow(int minCapacity) {
        // overflow-conscious code
        //原始容量大小
        int oldCapacity = buf.length;
        //新容量大小
        int newCapacity = oldCapacity << 1;
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
		// 复制buf到长度为newCapacity的新数组，然后赋值给buf
        buf = Arrays.copyOf(buf, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    //将指定字节写入到字节数组中
    public synchronized void write(int b) {
        ensureCapacity(count + 1);
        buf[count] = (byte) b;
        count += 1;
    }

    //从指定的字节数组off开始的len字节写入到这个字节数组输出流
    public synchronized void write(byte b[], int off, int len) {
        if ((off < 0) || (off > b.length) || (len < 0) ||
            ((off + len) - b.length > 0)) {
            throw new IndexOutOfBoundsException();
        }
        ensureCapacity(count + len);
        System.arraycopy(b, off, buf, count, len);
        count += len;
    }

    //将这个字节数组输出流的完整内容写入指定的输出流参数
    public synchronized void writeTo(OutputStream out) throws IOException {
        out.write(buf, 0, count);
    }

    //
    public synchronized void reset() {
        count = 0;
    }

    //创建一个新分配的字节数组
    public synchronized byte toByteArray()[] {
        return Arrays.copyOf(buf, count);
    }

    //返回buffer的大小
    public synchronized int size() {
        return count;
    }

    //toString方法
    public synchronized String toString() {
        return new String(buf, 0, count);
    }

    //toString方法
    public synchronized String toString(String charsetName)
        throws UnsupportedEncodingException
    {
        return new String(buf, 0, count, charsetName);
    }

    //
    @Deprecated
    public synchronized String toString(int hibyte) {
        return new String(buf, hibyte, 0, count);
    }

    //关闭流
    public void close() throws IOException {
    }

}

```

简单实例

```java
    @Test
    public void test2() throws Exception {
        File file = new File("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\FileIODemo.java");
        FileInputStream fileInputStream = new FileInputStream(file);
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        int len;
        while ((len = fileInputStream.read()) != -1) {
            byteArrayOutputStream.write(len);
        }
        String s = byteArrayOutputStream.toString();
        System.out.println(s);
        fileInputStream.close();
        byteArrayOutputStream.close();

    }
```

