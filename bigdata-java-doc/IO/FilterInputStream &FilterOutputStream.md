## FilterInputStream &FilterOutputStream

FilterInputStream是InputStream一个特殊的子类，而FilterOutputStream是OutputStream一个特殊的子类，这两个类都是采用了装饰设计模式。FilterInputStream内部包含了一个InputStream,而FilterOutputStream内部包含了一个OutputStream。使得FilterInputStream和FilterOutputStream在原有的基础上对InputStream和OutputStream进行了封装。

FilterInputStream的子类主要有BufferedInputStream,DataInputStream和PushbackInputStream三个类

FilterOutputStream的子类主要有BufferedOutputSream,DataOutputStream和PrintStream三个类

下面来看看相关的源码

### BufferedInputStream&BufferedOutputStream

BufferedInputStream提供了缓冲输入流功能，提供了一个缓冲数组，当调用read方法的时候，首先从缓冲区里面读取数据，如果缓冲区没有数据可读，则会从文件等物理数据源中读取数据放到缓冲区中,加快读取数据的速度。

```java
package java.io;
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

public
class BufferedInputStream extends FilterInputStream {

	//默认缓冲区大小，即8k
    private static int DEFAULT_BUFFER_SIZE = 8192;

    //能分配的最大的数组的大小
    private static int MAX_BUFFER_SIZE = Integer.MAX_VALUE - 8;

    //内部缓冲区数组
    protected volatile byte buf[];

    //原子更新器，保证对buf数组进行原子更新
    private static final
        AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
        AtomicReferenceFieldUpdater.newUpdater
        (BufferedInputStream.class,  byte[].class, "buf");

    //当前buf中的有效存储字节数,大小范围为0到buf.length
    protected int count;

    //缓冲区当前的位置，表示当前数据读取的位置
    protected int pos;

    //表示流中标记的位置，可通过reset方法返回至标记处
    protected int markpos = -1;

    //调用mark方法后在后续调用reset方法失败之前允许的最大预读量
    protected int marklimit;

    //检查输入流已经打开，并返回InputStream
    private InputStream getInIfOpen() throws IOException {
        //获取InputStream
        InputStream input = in;
        if (input == null)
            throw new IOException("Stream closed");
        return input;
    }

    //获得内部缓存数组buf，并对其进行检测，如果不为null，则返回该缓存数组buf
    private byte[] getBufIfOpen() throws IOException {
        byte[] buffer = buf;
        if (buffer == null)
            throw new IOException("Stream closed");
        return buffer;
    }

    //BufferedInputStream的构造函数
    public BufferedInputStream(InputStream in) {
        this(in, DEFAULT_BUFFER_SIZE);
    }

    //BufferedInputStream的构造函数
    public BufferedInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }

    //采用数据填充缓冲区,
    private void fill() throws IOException {
		//获取数组缓冲区
        byte[] buffer = getBufIfOpen();
		//在没有mark的情况下，pos为0
        if (markpos < 0)
            pos = 0;            /* no mark: throw away the buffer */
		//表示读取已经超过缓冲区中的位置，需要重新填充数据
        else if (pos >= buffer.length)  /* no room left in buffer */
			//表示流中存在标记，则将标记后的数据复制到缓冲区的头部，然后重新填充数据
            if (markpos > 0) {  /* can throw away early part of the buffer */
				// 把buffer中，markpos到pos的部分移动到0-sz处，pos设置为sz，markpos为0
				//获得当前位置到标记处的长度，通过System.arraycopy方法，将原缓存区中从标记处到当前位置的数据复制到新缓存的头部，将读取位置至于保存数据的尾部，标记处置为0。
                int sz = pos - markpos;
                System.arraycopy(buffer, markpos, buffer, 0, sz);
                pos = sz;
                markpos = 0;
			//buffer.length >= marklimit，即当缓存区的容量已经超过marklimit的限制时，便会将标记丢弃，直接将标记markpos重置为-1，直接将pos置为0，丢弃掉以前缓存区中的内容
            } else if (buffer.length >= marklimit) {
                markpos = -1;   /* buffer got too big, invalidate mark */
                pos = 0;        /* drop buffer contents */
			//如果缓存区的容量超过了最大限制(MAX_BUFFER_SIZE)，那么会抛出对应异常OutOfMemoryError。       
            } else if (buffer.length >= MAX_BUFFER_SIZE) {
                throw new OutOfMemoryError("Required array size too large");
            } else {            /* grow buffer */
                //建立一个长度为min(2*pos,marklimit,MAX_BUFFER_SIZE),的缓存数组，然后把原来0-pos移动到新数组的0-pos处
                int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                        pos * 2 : MAX_BUFFER_SIZE;
                if (nsz > marklimit)
                    nsz = marklimit;
                byte nbuf[] = new byte[nsz];
                System.arraycopy(buffer, 0, nbuf, 0, pos);
                //进行原子操作
                if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                    // Can't replace buf if there was an async close.
                    // Note: This would need to be changed if fill()
                    // is ever made accessible to multiple threads.
                    // But for now, the only way CAS can fail is via close.
                    // assert buf == null;
                    throw new IOException("Stream closed");
                }
                buffer = nbuf;
            }
        //
        count = pos;
        //从pos位置读取buffer.length - pos的字节到buffer数组中
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;
    }

    //用于从内置的缓冲区数组中读取数据，一次读取一个字节
    public synchronized int read() throws IOException {
		//如果pos>=count表示已经读取完毕
        if (pos >= count) {
			//读取数据到缓冲区字节数组buf中
            fill();
			//表示内部的输入流已经读完，返回-1
            if (pos >= count)
                return -1;
        }
        return getBufIfOpen()[pos++] & 0xff;
    }

    //一次读取多个字节的数据
    private int read1(byte[] b, int off, int len) throws IOException {
        int avail = count - pos;
        if (avail <= 0) {
            /* If the requested length is at least as large as the buffer, and
               if there is no mark/reset activity, do not bother to copy the
               bytes into the local buffer.  In this way buffered streams will
               cascade harmlessly. */
            if (len >= getBufIfOpen().length && markpos < 0) {
                return getInIfOpen().read(b, off, len);
            }
            fill();
            avail = count - pos;
            if (avail <= 0) return -1;
        }
        int cnt = (avail < len) ? avail : len;
        //对字节数组进行复制
        System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
        pos += cnt;
        return cnt;
    }

    //从这个字节输入流中读取字节到指定的字节数组中，从给定的off开始
    public synchronized int read(byte b[], int off, int len)
        throws IOException
    {
        getBufIfOpen(); // Check for closed stream
        if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

        int n = 0;
        for (;;) {
            int nread = read1(b, off + n, len - n);
            if (nread <= 0)
                return (n == 0) ? nread : n;
            n += nread;
            if (n >= len)
                return n;
            // if not closed but no bytes available, return
            InputStream input = in;
            if (input != null && input.available() <= 0)
                return n;
        }
    }

    //跳过指定的字节数
    public synchronized long skip(long n) throws IOException {
        getBufIfOpen(); // Check for closed stream
        if (n <= 0) {
            return 0;
        }
        long avail = count - pos;

        if (avail <= 0) {
            // If no mark position set then don't keep in buffer
            if (markpos <0)
                return getInIfOpen().skip(n);

            // Fill in buffer to save bytes for reset
            fill();
            avail = count - pos;
            if (avail <= 0)
                return 0;
        }

        long skipped = (avail < n) ? avail : n;
        pos += skipped;
        return skipped;
    }

    //返回当前的可用的字节数
    public synchronized int available() throws IOException {
        int n = count - pos;
        int avail = getInIfOpen().available();
        return n > (Integer.MAX_VALUE - avail)
                    ? Integer.MAX_VALUE
                    : n + avail;
    }

    //标记此输入流中的当前读取的位置，一般结合reset方法使用
    public synchronized void mark(int readlimit) {
        marklimit = readlimit;
        markpos = pos;
    }

   //将此流重新定位到最后一次在此输入流上调用mark方法时的位置,结合mark方法使用
    public synchronized void reset() throws IOException {
        getBufIfOpen(); // Cause exception if closed
        if (markpos < 0)
            throw new IOException("Resetting to invalid mark");
        pos = markpos;
    }

    //是否支持标记功能，true表示支持标记功能
    public boolean markSupported() {
        return true;
    }

    //用于关闭流
    public void close() throws IOException {
        byte[] buffer;
        while ( (buffer = buf) != null) {
            if (bufUpdater.compareAndSet(this, buffer, null)) {
                InputStream input = in;
                in = null;
                if (input != null)
                    input.close();
                return;
            }
            // Else retry in case a new buf was CASed in fill()
        }
    }
}

```

简单实例

```java
    @Test
    public void test1() throws Exception{

        BufferedInputStream bufferedInputStream=
                new BufferedInputStream(new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\SequenceInputDemo.java"),10);

        byte[] buffer = new byte[1024];

        int len;
        while ( (len = bufferedInputStream.read(buffer)) != -1) {
            System.out.print(new String(buffer));
            System.out.flush();
        }
        System.out.flush();
    }
```

BufferedOutputStream

BufferedOutputStream继承于FilterOutputStream，提供缓冲输出流功能。缓冲输出流相对于普通输出流的优势是，它提供了一个缓冲数组，只有缓冲数组满了或者手动flush时才会向磁盘写数据，避免频繁IO。核心思想是，提供一个缓冲数组，写入时首先操作缓冲数组。

```java
package java.io;


public
class BufferedOutputStream extends FilterOutputStream {
    //byte数组，用于存储数据的内部缓冲区
    protected byte buf[];

    //用于标识buf数组中存储的字节大小，范围为0到buf.length之间
    protected int count;

    //构造方法
    public BufferedOutputStream(OutputStream out) {
        this(out, 8192);
    }

    //构造方法
    public BufferedOutputStream(OutputStream out, int size) {
        super(out);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }

    //刷新内部缓冲区
    private void flushBuffer() throws IOException {
		//count>0标识内部缓存区中存在数据
        if (count > 0) {
			// out写入buf中，从0开始，count个字节
            out.write(buf, 0, count);
			//将count置为0
            count = 0;
        }
    }

    //将指定字节写入此缓冲输出流中
    public synchronized void write(int b) throws IOException {
		//如果缓冲区溢出，需要刷新缓冲区，同时将count值置为0
        if (count >= buf.length) {
            flushBuffer();
        }
        buf[count++] = (byte)b;
    }

    //将指定字节数组从off位置开始的len字节写入此缓冲输出流
    public synchronized void write(byte b[], int off, int len) throws IOException {
		//若果需要写入的长度大于内置缓存区中的容量，那么首先调用flushBuffer方法将缓存区中的数据写入，然后直接使用原始数据流将数据写入至目的地，避免了先写入内置缓存区中再从缓存区中写入目的地的步骤。
        if (len >= buf.length) {
            /* If the request length exceeds the size of the output buffer,
               flush the output buffer and then write the data directly.
               In this way buffered streams will cascade harmlessly. */
            flushBuffer();
            out.write(b, off, len);
            return;
        }
		//如果缓冲区溢出，需要刷新缓冲区，同时将count值置为0
        if (len > buf.length - count) {
            flushBuffer();
        }
		// 复制b的字节到buf中
        System.arraycopy(b, off, buf, count, len);
        count += len;
    }

    //刷新此缓冲区输出流
    public synchronized void flush() throws IOException {
        flushBuffer();
        out.flush();
    }
}

```

简单实例

```java
    @Test
    public void test2()throws Exception{
        byte[] buffer = new byte[1024];
        try (BufferedInputStream bis = new BufferedInputStream(
                new FileInputStream(new File("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\SequenceInputDemo.java")));
             BufferedOutputStream bos = new BufferedOutputStream(
                     new FileOutputStream(new File(
                             "C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\testcopy1.txt")))) {
            int len;
            while ( (len = bis.read(buffer)) != -1) {
                bos.write(buffer,0,len);
            }
            System.out.println("copying file has been finished..");
        }

    }
```

### DataInputStream&DataOutputStream

DataInputStream

DataInputStream数据输入流允许应用程序以机器无关的方式从底层输入流中读取基本的Java类型

```java
package java.io;

//用于读取二进制流字节并重建为任何java原始数据类型，也提供了一个转换为UTF-8编码的字符串类型方法
public
class DataInputStream extends FilterInputStream implements DataInput {

    //构造方法
    public DataInputStream(InputStream in) {
        super(in);
    }

    //byte数组
    private byte bytearr[] = new byte[80];
    //char数组
	private char chararr[] = new char[80];

    //从流中读取字节到b字节数组中，最终返回读取的字节数
    public final int read(byte b[]) throws IOException {
        return in.read(b, 0, b.length);
    }

    //从流中读取字节到b字节数组中，最终返回读取的字节数,off表示读取的起点、len表示读取的长度
    public final int read(byte b[], int off, int len) throws IOException {
        return in.read(b, off, len);
    }

    //从输入流中读取字节并存入到字节缓冲区b数组中
    public final void readFully(byte b[]) throws IOException {
        readFully(b, 0, b.length);
    }

    //从输入流中读取字节并存入到字节缓冲区b数组中，从off位置开始读取,读取len长度的字节
    public final void readFully(byte b[], int off, int len) throws IOException {
        if (len < 0)
            throw new IndexOutOfBoundsException();
        int n = 0;
        while (n < len) {
            int count = in.read(b, off + n, len - n);
            if (count < 0)
                throw new EOFException();
            n += count;
        }
    }

    //跳过n个字节的数据
    public final int skipBytes(int n) throws IOException {
        int total = 0;
        int cur = 0;

        while ((total<n) && ((cur = (int) in.skip(n-total)) > 0)) {
            total += cur;
        }

        return total;
    }

    //读取一个字节，如果这个字节不是0,则返回true,否则返回false;
    public final boolean readBoolean() throws IOException {
        int ch = in.read();
        if (ch < 0)
            throw new EOFException();
        return (ch != 0);
    }

    //读取并返回一个输入字节，该字节对应的值在-128到127之间
    public final byte readByte() throws IOException {
        int ch = in.read();
        if (ch < 0)
            throw new EOFException();
        return (byte)(ch);
    }

    //读取一个字节，返回无符号的int的值，值在0到255的范围之间
    public final int readUnsignedByte() throws IOException {
        int ch = in.read();
        if (ch < 0)
            throw new EOFException();
        return ch;
    }

    //读取两个输入字节，并返回一个short类型的值，如果读取的第一、第二字节分别为a,b 则返回为(short)((a << 8) | (b & 0xff))
    public final short readShort() throws IOException {
		//short占用两个字节，所以需要读取两个字节
        int ch1 = in.read();
        int ch2 = in.read();
        if ((ch1 | ch2) < 0)
            throw new EOFException();
        return (short)((ch1 << 8) + (ch2 << 0));
    }

    //读取两个输入字节，并返回一个无符号的short类型的值，如果读取的第一、第二字节分别为a,b 则返回为(((a & 0xff) << 8) | (b & 0xff))
    public final int readUnsignedShort() throws IOException {
		//short占用两个字节，所以需要读取两个字节
        int ch1 = in.read();
        int ch2 = in.read();
        if ((ch1 | ch2) < 0)
            throw new EOFException();
        return (ch1 << 8) + (ch2 << 0);
    }

    //读取两个输入字节，并返回一个char类型的值，如果读取的第一、第二字节分别为a,b 则返回为(char)((a << 8) | (b & 0xff))
    public final char readChar() throws IOException {
		//char占用两个字节，所以需要读取两个字节
        int ch1 = in.read();
        int ch2 = in.read();
        if ((ch1 | ch2) < 0)
            throw new EOFException();
        return (char)((ch1 << 8) + (ch2 << 0));
    }

    //读取四个字节并返回一个int类型的值，如果读取的第一、第二，第三，第四字节分别为a,b,c,d 则返回为(((a & 0xff) << 24) | ((b & 0xff) << 16) | ((c & 0xff) << 8) | (d & 0xff))
    public final int readInt() throws IOException {
		//int占用四个字节，所以需要读取四个字节
        int ch1 = in.read();
        int ch2 = in.read();
        int ch3 = in.read();
        int ch4 = in.read();
        if ((ch1 | ch2 | ch3 | ch4) < 0)
            throw new EOFException();
        return ((ch1 << 24) + (ch2 << 16) + (ch3 << 8) + (ch4 << 0));
    }

    private byte readBuffer[] = new byte[8];

    //读取八个字节并返回一个long数据，如a-h为读取到的八字节，结果会返回(((long)(a & 0xff) << 56) |  ((long)(b & 0xff) << 48) |  ((long)(c & 0xff) << 40) |  ((long)(d & 0xff) << 32) |  ((long)(e & 0xff) << 24) |  ((long)(f & 0xff) << 16) |  ((long)(g & 0xff) <<  8) | ((long)(h & 0xff)))
    public final long readLong() throws IOException {
		//long占用8个字节
        readFully(readBuffer, 0, 8);
        return (((long)readBuffer[0] << 56) +
                ((long)(readBuffer[1] & 255) << 48) +
                ((long)(readBuffer[2] & 255) << 40) +
                ((long)(readBuffer[3] & 255) << 32) +
                ((long)(readBuffer[4] & 255) << 24) +
                ((readBuffer[5] & 255) << 16) +
                ((readBuffer[6] & 255) <<  8) +
                ((readBuffer[7] & 255) <<  0));
    }

     //读取四个输入字节并返回一个float类型数据
    public final float readFloat() throws IOException {
        return Float.intBitsToFloat(readInt());
    }

     //读取八个如数字节并返回一个double数据类型
    public final double readDouble() throws IOException {
        return Double.longBitsToDouble(readLong());
    }

    private char lineBuffer[];

   //从输入流读取下一行文本，它读取连续字节，将每个字节分别转换为字符，直到遇到行终止符或文件结束符为止
    @Deprecated
    public final String readLine() throws IOException {
		//char数组
        char buf[] = lineBuffer;

        if (buf == null) {
            buf = lineBuffer = new char[128];
        }

        int room = buf.length;
        int offset = 0;
        int c;

loop:   while (true) {
            switch (c = in.read()) {
              case -1:
              case '\n':
                break loop;

              case '\r':
                int c2 = in.read();
                if ((c2 != '\n') && (c2 != -1)) {
                    if (!(in instanceof PushbackInputStream)) {
                        this.in = new PushbackInputStream(in);
                    }
                    ((PushbackInputStream)in).unread(c2);
                }
                break loop;

              default:
                if (--room < 0) {
                    buf = new char[offset + 128];
                    room = buf.length - offset - 1;
                    System.arraycopy(lineBuffer, 0, buf, 0, offset);
                    lineBuffer = buf;
                }
                buf[offset++] = (char) c;
                break;
            }
        }
        if ((c == -1) && (offset == 0)) {
            return null;
        }
        return String.copyValueOf(buf, 0, offset);
    }

    //读取一个字符串,使用UTF-8编码格式。这些字符将会作为字符串返回
    public final String readUTF() throws IOException {
        return readUTF(this);
    }

    //从流中以UTF编码格式读取数据，传入的参数为一个DataInput对象，用于对流进行读取操作。
    public final static String readUTF(DataInput in) throws IOException {
        int utflen = in.readUnsignedShort();
        byte[] bytearr = null;
        char[] chararr = null;
        if (in instanceof DataInputStream) {
            DataInputStream dis = (DataInputStream)in;
            if (dis.bytearr.length < utflen){
                dis.bytearr = new byte[utflen*2];
                dis.chararr = new char[utflen*2];
            }
            chararr = dis.chararr;
            bytearr = dis.bytearr;
        } else {
            bytearr = new byte[utflen];
            chararr = new char[utflen];
        }

        int c, char2, char3;
        int count = 0;
        int chararr_count=0;

        in.readFully(bytearr, 0, utflen);

        while (count < utflen) {
            c = (int) bytearr[count] & 0xff;
            if (c > 127) break;
            count++;
            chararr[chararr_count++]=(char)c;
        }

        while (count < utflen) {
            c = (int) bytearr[count] & 0xff;
            switch (c >> 4) {
                case 0: case 1: case 2: case 3: case 4: case 5: case 6: case 7:
                    /* 0xxxxxxx*/
                    count++;
                    chararr[chararr_count++]=(char)c;
                    break;
                case 12: case 13:
                    /* 110x xxxx   10xx xxxx*/
                    count += 2;
                    if (count > utflen)
                        throw new UTFDataFormatException(
                            "malformed input: partial character at end");
                    char2 = (int) bytearr[count-1];
                    if ((char2 & 0xC0) != 0x80)
                        throw new UTFDataFormatException(
                            "malformed input around byte " + count);
                    chararr[chararr_count++]=(char)(((c & 0x1F) << 6) |
                                                    (char2 & 0x3F));
                    break;
                case 14:
                    /* 1110 xxxx  10xx xxxx  10xx xxxx */
                    count += 3;
                    if (count > utflen)
                        throw new UTFDataFormatException(
                            "malformed input: partial character at end");
                    char2 = (int) bytearr[count-2];
                    char3 = (int) bytearr[count-1];
                    if (((char2 & 0xC0) != 0x80) || ((char3 & 0xC0) != 0x80))
                        throw new UTFDataFormatException(
                            "malformed input around byte " + (count-1));
                    chararr[chararr_count++]=(char)(((c     & 0x0F) << 12) |
                                                    ((char2 & 0x3F) << 6)  |
                                                    ((char3 & 0x3F) << 0));
                    break;
                default:
                    /* 10xx xxxx,  1111 xxxx */
                    throw new UTFDataFormatException(
                        "malformed input around byte " + count);
            }
        }
        // The number of chars produced may be less than utflen
        return new String(chararr, 0, chararr_count);
    }
}

```

DataOutputStream

DataOutputStream数据输出流允许应用程序将基本Java数据类型写到基础输出流

```java
package java.io;


public
class DataOutputStream extends FilterOutputStream implements DataOutput {
    //用于记录实际写入数据的字节数，当写入的数据超过int型数据的表达范围时，该值将被赋予Integer.MAX_VALUE.
    protected int written;

    //字节数组，用于缓冲字节
    private byte[] bytearr = null;

    //构造方法
    public DataOutputStream(OutputStream out) {
        super(out);
    }

    //用于增加written，记录流中多少字节的数据，如果长度超过了int型表达范围，则该值一直为Integer.MAX_VALUE
    private void incCount(int value) {
        int temp = written + value;
        if (temp < 0) {
            temp = Integer.MAX_VALUE;
        }
        written = temp;
    }

    //取传入的int行数据的低8位数据写入到输出流中，高24位忽略
    public synchronized void write(int b) throws IOException {
        out.write(b);
        incCount(1);
    }

    //将传入的字节数组从off位置开始往后len长度的数据写入到输出流
    public synchronized void write(byte b[], int off, int len)
        throws IOException
    {
        out.write(b, off, len);
        incCount(len);
    }

    //用于将缓存区中的数据写入流中。
    public void flush() throws IOException {
        out.flush();
    }

    //该方法向输出流中写入一个boolean类型的值，如果传入的boolean型参数v是true，那么向流中写入1，如果传入的boolean型参数v是false，那么向流中写入0
    public final void writeBoolean(boolean v) throws IOException {
        out.write(v ? 1 : 0);
        incCount(1);
    }

     //该方法将截取传入参数的低八位写入输出流中，高24位将被忽略
    public final void writeByte(int v) throws IOException {
        out.write(v);
        incCount(1);
    }

    //向输出流中写入两个字节的数据，通过(byte)(0xff & (v >> 8))和(byte)(0xff & v)两个小操作来分别得到需要写入的数据，即int型数据后16位中的高8位和低八位.通过该方法向输出流中写入的Short型数据，需要通过DataInput接口中的readShort方法来获取
    public final void writeShort(int v) throws IOException {
        out.write((v >>> 8) & 0xFF);
        out.write((v >>> 0) & 0xFF);
        incCount(2);
    }

    //该方法向输出流中写入一个包含两个字节的char型数据，通过(byte)(0xff & (v >> 8))和(byte)(0xff & v)两个小操作来得到需要写入的数据。通过该方法向输出流中写入的char型数据
    public final void writeChar(int v) throws IOException {
        out.write((v >>> 8) & 0xFF);
        out.write((v >>> 0) & 0xFF);
        incCount(2);
    }

    //该方法向输出流中写入一个包含四个字节的int型数据，通过(byte)(0xff & (v >> 24))、(byte)(Oxff & (v >> 16))、(byte)(Oxff & (v >> 8))和(byte)(Oxff & v)四个操作来得到需要写入的数据
    public final void writeInt(int v) throws IOException {
        out.write((v >>> 24) & 0xFF);
        out.write((v >>> 16) & 0xFF);
        out.write((v >>>  8) & 0xFF);
        out.write((v >>>  0) & 0xFF);
        incCount(4);
    }

    private byte writeBuffer[] = new byte[8];

       //该方法向输出流中写入一个包含八个字节的long型数据，通过(byte)(0xff & (v >> 56))、(byte)(0xff & (v >> 48))、(byte)(0xff & (v >> 40))、(byte)(0xff & (v >> 32))、(byte)(0xff & (v >> 24))、(byte)(0xff & (v >> 16))、(byte)(0xff & (v >>  8))和(byte)(0xff & v)八个小操作来得到需要写入数据。通过该方法向输出流中写入的int型数据
    public final void writeLong(long v) throws IOException {
        writeBuffer[0] = (byte)(v >>> 56);
        writeBuffer[1] = (byte)(v >>> 48);
        writeBuffer[2] = (byte)(v >>> 40);
        writeBuffer[3] = (byte)(v >>> 32);
        writeBuffer[4] = (byte)(v >>> 24);
        writeBuffer[5] = (byte)(v >>> 16);
        writeBuffer[6] = (byte)(v >>>  8);
        writeBuffer[7] = (byte)(v >>>  0);
        out.write(writeBuffer, 0, 8);
        incCount(8);
    }

   //向流中写入一个包含4个字节的float型数据
    public final void writeFloat(float v) throws IOException {
        writeInt(Float.floatToIntBits(v));
    }

    //向流中写入一个包含8个字节的double型数据
    public final void writeDouble(double v) throws IOException {
        writeLong(Double.doubleToLongBits(v));
    }

    //用于将传入String类型s中的每个字符写入输出流中
    public final void writeBytes(String s) throws IOException {
        int len = s.length();
        for (int i = 0 ; i < len ; i++) {
            out.write((byte)s.charAt(i));
        }
        incCount(len);
    }

    //将传入String类型s中的每个字符写入输出流中
    public final void writeChars(String s) throws IOException {
        int len = s.length();
        for (int i = 0 ; i < len ; i++) {
            int v = s.charAt(i);
            out.write((v >>> 8) & 0xFF);
            out.write((v >>> 0) & 0xFF);
        }
        incCount(len * 2);
    }

    //该方法将传入的字符串s以utf-8的编码格式写入输出流中，通过该方法向输出流中写入的数据可以通过DataInput接口中的readUTF来进行读取。
    public final void writeUTF(String str) throws IOException {
        writeUTF(str, this);
    }

    //依次向流中以utf8的格式写入一个String类型的数据。
    static int writeUTF(String str, DataOutput out) throws IOException {
		//字符串长度
        int strlen = str.length();
        int utflen = 0;
        int c, count = 0;

        /* use charAt instead of copying String to char array */
        for (int i = 0; i < strlen; i++) {
            c = str.charAt(i);
            if ((c >= 0x0001) && (c <= 0x007F)) {
                utflen++;
            } else if (c > 0x07FF) {
                utflen += 3;
            } else {
                utflen += 2;
            }
        }

        if (utflen > 65535)
            throw new UTFDataFormatException(
                "encoded string too long: " + utflen + " bytes");

        byte[] bytearr = null;
        if (out instanceof DataOutputStream) {
            DataOutputStream dos = (DataOutputStream)out;
            if(dos.bytearr == null || (dos.bytearr.length < (utflen+2)))
                dos.bytearr = new byte[(utflen*2) + 2];
            bytearr = dos.bytearr;
        } else {
            bytearr = new byte[utflen+2];
        }

        bytearr[count++] = (byte) ((utflen >>> 8) & 0xFF);
        bytearr[count++] = (byte) ((utflen >>> 0) & 0xFF);

        int i=0;
        for (i=0; i<strlen; i++) {
           c = str.charAt(i);
           if (!((c >= 0x0001) && (c <= 0x007F))) break;
           bytearr[count++] = (byte) c;
        }

        for (;i < strlen; i++){
            c = str.charAt(i);
            if ((c >= 0x0001) && (c <= 0x007F)) {
                bytearr[count++] = (byte) c;

            } else if (c > 0x07FF) {
                bytearr[count++] = (byte) (0xE0 | ((c >> 12) & 0x0F));
                bytearr[count++] = (byte) (0x80 | ((c >>  6) & 0x3F));
                bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
            } else {
                bytearr[count++] = (byte) (0xC0 | ((c >>  6) & 0x1F));
                bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
            }
        }
        out.write(bytearr, 0, utflen+2);
        return utflen + 2;
    }

    //返回当前写入流中数据的字节总数。
    public final int size() {
        return written;
    }
}

```

### PushbackInputStream

PushbackInputStream流提供流一个回推功能，unread方法。实现逻辑是内部提供一个数组用来存储回推的数据，其实就是把要回推的数据写入这个数组中，下次读取的时候从这个数组中读取数据，而不是真是把数据写回去，对于原始流没有影响，这也是装饰器模式的优点。

```java
package java.io;

//提供具有回推功能的输入流
public
class PushbackInputStream extends FilterInputStream {
    //字节数组，作为流中读取数据的临时缓存，回推功能的实现主要依赖这个临时缓存数组
    protected byte[] buf;

    //pushback缓冲区中下一次读取字节的位置
    protected int pos;

    //确保流没有关闭
    private void ensureOpen() throws IOException {
        if (in == null)
            throw new IOException("Stream closed");
    }

    //构造方法
    public PushbackInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("size <= 0");
        }
        this.buf = new byte[size];
        this.pos = size;
    }

    //构造方法
    public PushbackInputStream(InputStream in) {
        this(in, 1);
    }

    //每次读取一个字节的数据，优先读取缓存中的数据
    public int read() throws IOException {
        ensureOpen();
		//pos < buf.length则优先从回推缓存中读取数据
        if (pos < buf.length) {
            return buf[pos++] & 0xff;
        }
        return super.read();
    }

    //从off位置开始，读取len长度的字节到b字节数组中，该方法首先会读取buf临时缓存中的字节数据
    public int read(byte[] b, int off, int len) throws IOException {
		//确保流没有关闭
        ensureOpen();
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }
		//临时缓存中是否有可用的字节
        int avail = buf.length - pos;
        if (avail > 0) {
            if (len < avail) {
                avail = len;
            }
			//进行字节数组复制操作
            System.arraycopy(buf, pos, b, off, avail);
            pos += avail;
            off += avail;
            len -= avail;
        }
		//len>0表示还需要向字节数组中传入数据
        if (len > 0) {
            len = super.read(b, off, len);
            if (len == -1) {
                return avail == 0 ? -1 : avail;
            }
            return avail + len;
        }
        return avail;
    }

    //一次可以回推一个字节数据的unread方法，将b存入到临时的缓冲区中,该方法返回后，下一个要读取的字节的值(字节)为b
    public void unread(int b) throws IOException {
        ensureOpen();
        if (pos == 0) {
            throw new IOException("Push back buffer is full");
        }
		//把数据放入到临时缓冲区中，并将pos值减一
        buf[--pos] = (byte)b;
    }

    //一次回推多个字节的方法，b为存放的要回推的操作的字节数组，off和len分别代表从传入的数组中取出数据的起点，以及其需要回推的数据长度
    public void unread(byte[] b, int off, int len) throws IOException {
        ensureOpen();
        if (len > pos) {
            throw new IOException("Push back buffer is full");
        }
        pos -= len;
		//利用System.arraycopy方法将指定的数据拷贝到内置的回推缓存区之中。
        System.arraycopy(b, off, buf, pos, len);
    }

    //一次可以回推多个字节数据的unread方法
    public void unread(byte[] b) throws IOException {
        unread(b, 0, b.length);
    }

    //返回当前流中可以读取的数据总量
    public int available() throws IOException {
        ensureOpen();
        int n = buf.length - pos;
        int avail = super.available();
        return n > (Integer.MAX_VALUE - avail)
                    ? Integer.MAX_VALUE
                    : n + avail;
    }

    //跳过指定的字节数，最后返回实际跳过的字节数
    public long skip(long n) throws IOException {
        ensureOpen();
        if (n <= 0) {
            return 0;
        }

        long pskip = buf.length - pos;
        if (pskip > 0) {
            if (n < pskip) {
                pskip = n;
            }
            pos += pskip;
            n -= pskip;
        }
        if (n > 0) {
            pskip += super.skip(n);
        }
        return pskip;
    }

    //不支持标记功能
    public boolean markSupported() {
        return false;
    }

    
    public synchronized void mark(int readlimit) {
    }

    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }

    //关闭流
    public synchronized void close() throws IOException {
        if (in == null)
            return;
        in.close();
        in = null;
        buf = null;
    }
}

```

简单实例

```java
    @Test
    public void test1() throws Exception {
        String s = "abcdefg";

        ByteArrayInputStream inputStream = new ByteArrayInputStream(s.getBytes(StandardCharsets.UTF_8));
        PushbackInputStream pushbackInputStream = new PushbackInputStream(inputStream);
        int n;

        while ((n = pushbackInputStream.read()) != -1) {
            System.out.print((char) n);
            if ('b' == n) {
                pushbackInputStream.unread('U');
            }
        }


    }
```

### PrintStream

PrintStream向另一个输出流添加功能，即方便地打印各种数据值的表示的能力，与其他outputstream不同，PrintStream从不抛出IOException，因为做了try{}catch(){}会将异常捕获,出现异常情况会在内部设置标识,通过checkError()获取此标识.PrintStream打印的所有字符都使用平台的默认字符编码转换为字节。

```java
package java.io;

import java.util.Formatter;
import java.util.Locale;
import java.nio.charset.Charset;
import java.nio.charset.IllegalCharsetNameException;
import java.nio.charset.UnsupportedCharsetException;

public class PrintStream extends FilterOutputStream
    implements Appendable, Closeable
{
	//是否自动刷新的标识
    private final boolean autoFlush;
	//类内部是否报错的标志
    private boolean trouble = false;
	//格式化对象Formatter，用于数据格式化
    private Formatter formatter;

    //同时跟踪文本输出流和字符输出流，在不刷新整个流的情况下刷新它们的缓冲区
    private BufferedWriter textOut;
    private OutputStreamWriter charOut;

    //进行判断不为null,否则抛出异常
    private static <T> T requireNonNull(T obj, String message) {
        if (obj == null)
            throw new NullPointerException(message);
        return obj;
    }

    //返回给定的字符集
    private static Charset toCharset(String csn)
        throws UnsupportedEncodingException
    {
        requireNonNull(csn, "charsetName");
        try {
            return Charset.forName(csn);
        } catch (IllegalCharsetNameException|UnsupportedCharsetException unused) {
            // UnsupportedEncodingException should be thrown
            throw new UnsupportedEncodingException(csn);
        }
    }

    //构造方法
    private PrintStream(boolean autoFlush, OutputStream out) {
        super(out);
        this.autoFlush = autoFlush;
        this.charOut = new OutputStreamWriter(this);
        this.textOut = new BufferedWriter(charOut);
    }
	//构造方法
    private PrintStream(boolean autoFlush, OutputStream out, Charset charset) {
        super(out);
        this.autoFlush = autoFlush;
        this.charOut = new OutputStreamWriter(this, charset);
        this.textOut = new BufferedWriter(charOut);
    }

    //构造方法
    private PrintStream(boolean autoFlush, Charset charset, OutputStream out)
        throws UnsupportedEncodingException
    {
        this(autoFlush, out, charset);
    }

    //构造方法,字节输出流out作为PrintStream流的输出流，默认为不自动刷新
    public PrintStream(OutputStream out) {
        this(out, false);
    }

    //构造方法，字节输出流out作为PrintStream流的输出流，根据传入的autoFlush确定是否自动刷新,true为自动刷新，否则不自动刷新
    public PrintStream(OutputStream out, boolean autoFlush) {
        this(autoFlush, requireNonNull(out, "Null output stream"));
    }

    //构造方法，指定编码名称encoding的PrintStream，字节输出流out作为PrintStream流的输出流，根据传入的autoFlush确定是否自动刷新,true为自动刷新，否则不自动刷新
    public PrintStream(OutputStream out, boolean autoFlush, String encoding)
        throws UnsupportedEncodingException
    {
        this(autoFlush,
             requireNonNull(out, "Null output stream"),
             toCharset(encoding));
    }

    //构造方法,文件名称的PrintStream流，字节输出流FileOutputStream作为PrintStream流的输出流，默认为不自动刷新
    public PrintStream(String fileName) throws FileNotFoundException {
        this(false, new FileOutputStream(fileName));
    }

    //构造方法,文件名称和字符编码名称csn的PrintStream流，字节输出流FileOutputStream作为PrintStream流的输出流，默认为不自动刷新
    public PrintStream(String fileName, String csn)
        throws FileNotFoundException, UnsupportedEncodingException
    {
        // ensure charset is checked before the file is opened
        this(false, toCharset(csn), new FileOutputStream(fileName));
    }

    //构造方法
    public PrintStream(File file) throws FileNotFoundException {
        this(false, new FileOutputStream(file));
    }

    //构造方法
    public PrintStream(File file, String csn)
        throws FileNotFoundException, UnsupportedEncodingException
    {
        // ensure charset is checked before the file is opened
        this(false, toCharset(csn), new FileOutputStream(file));
    }

    //确保流没有关闭
    private void ensureOpen() throws IOException {
        if (out == null)
            throw new IOException("Stream closed");
    }

    //对流进行刷新
    public void flush() {
        synchronized (this) {
            try {
                ensureOpen();
				//对底层的out进行flush
                out.flush();
            }
            catch (IOException x) {
                trouble = true;
            }
        }
    }

    private boolean closing = false; /* To avoid recursive closing */

    //关闭流
    public void close() {
        synchronized (this) {
            if (! closing) {
                closing = true;
                try {
                    textOut.close();
					
                    out.close();
                }
                catch (IOException x) {
                    trouble = true;
                }
                textOut = null;
                charOut = null;
                out = null;
            }
        }
    }

    //检查流中异常状态,如果PrintStream流中有异常抛出,返回true
    public boolean checkError() {
        if (out != null)
            flush();
        if (out instanceof java.io.PrintStream) {
            PrintStream ps = (PrintStream) out;
            return ps.checkError();
        }
        return trouble;
    }

    //将流的错误状态设置为true
    protected void setError() {
        trouble = true;
    }

    //清除此流的内部错误状态。
    protected void clearError() {
        trouble = false;
    }

    //将单个字节b写到PrintStream流中
    public void write(int b) {
        try {
            synchronized (this) {
                ensureOpen();
				//底层采用的是out进行写入
                out.write(b);
				//遇到\n且autoFlush为true情况下进行刷新
                if ((b == '\n') && autoFlush)
                    out.flush();
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }

    //将字节数组buf中off位置开始,len个字节写到PrintStream流中.
    public void write(byte buf[], int off, int len) {
        try {
            synchronized (this) {
                ensureOpen();
                out.write(buf, off, len);
                if (autoFlush)
                    out.flush();
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }

	//将字符数组buf写到PrintStream流中
    private void write(char buf[]) {
        try {
            synchronized (this) {
                ensureOpen();
                textOut.write(buf);
                textOut.flushBuffer();
                charOut.flushBuffer();
                if (autoFlush) {
                    for (int i = 0; i < buf.length; i++)
                        if (buf[i] == '\n')
                            out.flush();
                }
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }

	//将字符串s写到PrintStream流中
    private void write(String s) {
        try {
            synchronized (this) {
                ensureOpen();
                textOut.write(s);
                textOut.flushBuffer();
                charOut.flushBuffer();
                if (autoFlush && (s.indexOf('\n') >= 0))
                    out.flush();
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }
	//将换行符写到PrintStream流中
    private void newLine() {
        try {
            synchronized (this) {
                ensureOpen();
                textOut.newLine();
                textOut.flushBuffer();
                charOut.flushBuffer();
                if (autoFlush)
                    out.flush();
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }

    //将boolean类型数据对应的字符串"true"或者"false"写到PrintStream流中,实际调用write()方法
    public void print(boolean b) {
        write(b ? "true" : "false");
    }

    //将单个字符写到PrintStream流中
    public void print(char c) {
        write(String.valueOf(c));
    }

    //将int类型转换为字符串类型写到PrintStream流中
    public void print(int i) {
        write(String.valueOf(i));
    }

    //将long类型转换为字符串类型写到PrintStream流中
    public void print(long l) {
        write(String.valueOf(l));
    }

     //将float类型转换为字符串类型写到PrintStream流中
    public void print(float f) {
        write(String.valueOf(f));
    }

     //将double类型转换为字符串类型写到PrintStream流中
    public void print(double d) {
        write(String.valueOf(d));
    }

    //将字符数组写到PrintStream流中
    public void print(char s[]) {
        write(s);
    }

    //将字符串写到PrintStream流中
    public void print(String s) {
        if (s == null) {
            s = "null";
        }
        write(s);
    }

    //将Object对象写到PrintStream流中
    public void print(Object obj) {
        write(String.valueOf(obj));
    }


    /* 下面的方法同print类似，只不过会进行换行*/


    public void println() {
        newLine();
    }

    public void println(boolean x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(char x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(int x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(long x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(float x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(double x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(char x[]) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(String x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }

    public void println(Object x) {
        String s = String.valueOf(x);
        synchronized (this) {
            print(s);
            newLine();
        }
    }


    //使用默认的Locale以及指定的格式字符串和参数将格式化字符串写入输出流
    public PrintStream printf(String format, Object ... args) {
        return format(format, args);
    }

    //使用指定的Locale以及格式字符串和参数将格式化字符串写入输出流
    public PrintStream printf(Locale l, String format, Object ... args) {
        return format(l, format, args);
    }

    //根据默认的Locale值和format格式来格式化数据args写到PrintStream输出流中
    public PrintStream format(String format, Object ... args) {
        try {
            synchronized (this) {
                ensureOpen();
                if ((formatter == null)
                    || (formatter.locale() != Locale.getDefault()))
                    formatter = new Formatter((Appendable) this);
                formatter.format(Locale.getDefault(), format, args);
            }
        } catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        } catch (IOException x) {
            trouble = true;
        }
        return this;
    }

    //据Locale值和format格式进行格式化后写到PrintStream输出流中.
    public PrintStream format(Locale l, String format, Object ... args) {
        try {
            synchronized (this) {
                ensureOpen();
                if ((formatter == null)
                    || (formatter.locale() != l))
                    formatter = new Formatter(this, l);
                formatter.format(l, format, args);
            }
        } catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        } catch (IOException x) {
            trouble = true;
        }
        return this;
    }

    //将指定的字符序列追加到此输出流
    public PrintStream append(CharSequence csq) {
        if (csq == null)
            print("null");
        else
            print(csq.toString());
        return this;
    }

    //将指定的字符序列追加到此输出流,start、end分别代表此字符序列的开始位置和结束位置
    public PrintStream append(CharSequence csq, int start, int end) {
        CharSequence cs = (csq == null ? "null" : csq);
        write(cs.subSequence(start, end).toString());
        return this;
    }

    //将指定的字符添加到此输出流中
    public PrintStream append(char c) {
        print(c);
        return this;
    }

}

```

简单实例

```java
    @Test
    public void test1() {
        PrintStream out = System.out;
        out.println("hello java");
        out.println(new Object());
    }


    @Test
    public void test2() throws Exception{
        PrintStream printStream=new PrintStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\test1.txt");

        printStream.println("hello java");
        printStream.println("hello java IO");

    }
```

