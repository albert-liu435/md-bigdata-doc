### java I/O

I/O是Input和Output的缩写，I/O是用来处理设备之间的数据读写问题的。如读写文件，网络通信等。同时Input和Output是相对于内存来说的，input用于读取外部数据到程序(内存)中，output用于将程序(内存)中的数据输出到外部(磁盘，网络等)。

java中数据的输入与输出都是以流的方式进行，java.io包下提供了各种"流"类和接口，用以获取不同种类的数据，并通过标准的方法输入或输出数据。

按照数据单位不同分为字节流和字符流，

字节流：字节流操作的基本单位为byte，通常用来操作视频，图片等文件信息

字符流：字符流操作的基本单位为char(字符),通常用来操作文本内容

按照流的方向分为输入流和输出流。

### java IO体系

字节输入流

![ByteArrayInputStream](.\pic\ByteArrayInputStream.png)

字节输出流

![OutputStream](.\pic\OutputStream.png)

字符输入流

![Reader](.\pic\Reader.png)

字符输出流

![Writer](.\pic\Writer.png)

### 字节流

字节输出流OutputStream

```java

package java.io;

//字节输出流所有类的基类
public abstract class OutputStream implements Closeable, Flushable {
    //将一个字节写入到此输出流中,写入的为b的低8位，而高24位会进行忽略
    public abstract void write(int b) throws IOException;

    //将一个字节数组写入到此输出流中
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }

    //从b的off位置开始，获取len个字节数据，写到此输出流中
    public void write(byte b[], int off, int len) throws IOException {
        //b为null 抛出异常
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        //循环将字节数组写入到输出流中
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }

    //对输出流进行刷新，则缓冲区中的字节会被全部刷新
    public void flush() throws IOException {
    }

    //关闭输出流
    public void close() throws IOException {
    }

}
```

字节输入流

```java


package java.io;

//字节输入流所有类的基类
public abstract class InputStream implements Closeable {

    //跳过的最大的字节数
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;

    //从流中读取一个字节并返回，如果到达内容末尾则返回-1，1byte=8bit
    public abstract int read() throws IOException;

    //从流中读取一定数量的字节并将其存储到缓冲区数组b中，并返回读取的字节个数，若是返回-1表示到了结尾，没有数据
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }

    //从流中off位置开始读取len个字节的数据到b数组中，并返回读取的字节个数，若是返回-1表示到了结尾，没有数据
    public int read(byte b[], int off, int len) throws IOException {
        //字节数组为null,则抛出异常
        if (b == null) {
            throw new NullPointerException();
            //off < 0 || len < 0 || len > b.length - off 的情况下抛出数组越界
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }
		//读取字节
        int c = read();
        //判断是否到达末尾
        if (c == -1) {
            return -1;
        }
        //将读取的字节放入到字节数组中
        b[off] = (byte)c;

        int i = 1;
        try {
            //循环读取数据放入到字节数组中，最多读取为len次
            for (; i < len ; i++) {
                c = read();
                if (c == -1) {
                    break;
                }
                b[off + i] = (byte)c;
            }
        } catch (IOException ee) {
        }
        return i;
    }

    //跳过n长度的字节
    public long skip(long n) throws IOException {
		//跳过的字节的数量
        long remaining = n;
        int nr;
		//表示跳过的字节数量为0，相当于对流中的数据不进行丢弃
        if (n <= 0) {
            return 0;
        }
		//丢弃的字节最大值为  MAX_SKIP_BUFFER_SIZE和remaining的最小值
        int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
        byte[] skipBuffer = new byte[size];
        while (remaining > 0) {
            //将流中的数据读取到skipBuffer缓冲区中
            nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
            if (nr < 0) {
                break;
            }
            remaining -= nr;
        }
		//返回实际跳过的字节数量
        return n - remaining;
    }

    //返回可读的字节数
    public int available() throws IOException {
        return 0;
    }

	//关闭流
    public void close() throws IOException {}

	//标记此输入流中的当前读取的位置，一般结合reset方法使用
    public synchronized void mark(int readlimit) {}

	//将此流重新定位到最后一次在此输入流上调用mark方法时的位置,结合mark方法使用
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }

	//该输入流是否支持标记功能，即是否支持mark和reset方法
    public boolean markSupported() {
        return false;
    }

}
```

### 字符流

Reader字符输入流

```java

package java.io;

//字符流输入流的抽象基类
public abstract class Reader implements Readable, Closeable {

    //锁,用于同步操作
    protected Object lock;

    //构造方法
    protected Reader() {
        this.lock = this;
    }

    //构造方法
    protected Reader(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }

    //将字符读入到指定的字符缓冲区，CharBuffer为nio包下面的类
    public int read(java.nio.CharBuffer target) throws IOException {
        //target剩余空间的长度
        int len = target.remaining();
        //创建一个字符数组
        char[] cbuf = new char[len];
       //将数据读入到字符数组中，并返回读取的长度
        int n = read(cbuf, 0, len);
        if (n > 0)
            //cbuf往target填充数据
            target.put(cbuf, 0, n);
        return n;
    }

    //读取一个字符，返回一个结果int,若是到达流的末尾，返回-1
    public int read() throws IOException {
        char cb[] = new char[1];
        if (read(cb, 0, 1) == -1)
            return -1;
        else
            return cb[0];
    }

    //将字符读入cbuf数组中
    public int read(char cbuf[]) throws IOException {
        return read(cbuf, 0, cbuf.length);
    }

      //从流中off位置开始读取len个字符的数据到cbuf数组中，并返回读取的字符个数，若是返回-1表示到了结尾，没有数据
    abstract public int read(char cbuf[], int off, int len) throws IOException;

    //最大的跳跃的缓存大小
    private static final int maxSkipBufferSize = 8192;

    //跳跃缓冲区
    private char skipBuffer[] = null;

    //跳过的字符
    public long skip(long n) throws IOException {
        //小于0就抛出异常
        if (n < 0L)
            throw new IllegalArgumentException("skip value is negative");
        //最大跳过的为 n与maxSkipBufferSize取最小值
        int nn = (int) Math.min(n, maxSkipBufferSize);
        //进行同步操作
        synchronized (lock) {
            if ((skipBuffer == null) || (skipBuffer.length < nn))
                skipBuffer = new char[nn];
            long r = n;
            // r为还要跳过的字符数
            while (r > 0) {
                // 通过调用read方法，读入skipBuffer来跳过
                int nc = read(skipBuffer, 0, (int)Math.min(r, nn));
                // 如果某次读取数为-1，跳出循环
                if (nc == -1)
                    break;
                r -= nc;
            }
            return n - r;
        }
    }

    //该流是否已经准备好
    public boolean ready() throws IOException {
        return false;
    }

    //是否支持标记，默认不支持
    public boolean markSupported() {
        return false;
    }

    //标记流中当前的位置
    public void mark(int readAheadLimit) throws IOException {
        throw new IOException("mark() not supported");
    }

    //重置。如果流已被标记，则尝试在标记处重新定位它。
    public void reset() throws IOException {
        throw new IOException("reset() not supported");
    }

    //关闭流
     abstract public void close() throws IOException;

}
```

Writer字符输出流

```java

package java.io;


//写入字符流的所有类的基类
public abstract class Writer implements Appendable, Closeable, Flushable {

    //临时缓冲区，用来保存单个字符
    private char[] writeBuffer;

    //writeBuffer的大小
    private static final int WRITE_BUFFER_SIZE = 1024;

    //锁，用来进行同步
    protected Object lock;
    //构造方法
    protected Writer() {
        this.lock = this;
    }
	//构造方法
    protected Writer(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }

    //写入一个字符
    public void write(int c) throws IOException {
        //进行同步作用
        synchronized (lock) {
            if (writeBuffer == null){
                writeBuffer = new char[WRITE_BUFFER_SIZE];
            }
			//先写入writeBuffer，然后再把writeBuffer写入到流中
            writeBuffer[0] = (char) c;
            write(writeBuffer, 0, 1);
        }
    }

    //写入一个字符数组都流中
    public void write(char cbuf[]) throws IOException {
        write(cbuf, 0, cbuf.length);
    }

    //写入一个字符数组的特定字符到流中
    abstract public void write(char cbuf[], int off, int len) throws IOException;

    //写入字符串到流中
    public void write(String str) throws IOException {
        write(str, 0, str.length());
    }
	//写入字符串的一部分，其中从字符串的off位置开始，取len长度的字符串
    public void write(String str, int off, int len) throws IOException {
        synchronized (lock) {
            char cbuf[];
            if (len <= WRITE_BUFFER_SIZE) {
                if (writeBuffer == null) {
                    writeBuffer = new char[WRITE_BUFFER_SIZE];
                }
                cbuf = writeBuffer;
            } else {    // Don't permanently allocate very large buffers.
                cbuf = new char[len];
            }
            str.getChars(off, (off + len), cbuf, 0);
            write(cbuf, 0, len);
        }
    }

    //向写入器追加指定的字符序列
    public Writer append(CharSequence csq) throws IOException {
        if (csq == null)
            write("null");
        else
            write(csq.toString());
        return this;
    }

    //向写入器追加指定字符序列的子序列。
    public Writer append(CharSequence csq, int start, int end) throws IOException {
        CharSequence cs = (csq == null ? "null" : csq);
        write(cs.subSequence(start, end).toString());
        return this;
    }

    //追加单个字符
    public Writer append(char c) throws IOException {
        write(c);
        return this;
    }

    //刷新流
    abstract public void flush() throws IOException;

    //关闭流
    abstract public void close() throws IOException;

}

```

