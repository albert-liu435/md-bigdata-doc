CharArrayReader&CharArrayWriter

### CharArrayReader

字符数组输入流，与ByteArrayInputStream类似,它包含一个内部缓冲区，该缓冲区包含从流中读取的字符。内部额外的定义了一个计数器，它被用来跟踪 read() 方法要读取的下一个字符。

```java
package java.io;


public class CharArrayReader extends Reader {
    //字符缓冲，用于存储从流中读取到的字符
    protected char buf[];

    //当前缓冲区的位置
    protected int pos;

    //被标记的字符位置，盗用rest方法是会回到该位置
    protected int markedPos = 0;

    //字符缓冲区数组的大小，即buf数组的末尾的索引
    protected int count;

    //构造方法，根据给定的字符数组创建CharArrayReader
    public CharArrayReader(char buf[]) {
        this.buf = buf;
        this.pos = 0;
        this.count = buf.length;
    }

    //根据给定的字符数组创建CharArrayReader，offset为开始读取的索引，length是读取的长度
    public CharArrayReader(char buf[], int offset, int length) {
        if ((offset < 0) || (offset > buf.length) || (length < 0) ||
            ((offset + length) < 0)) {
            throw new IllegalArgumentException();
        }
        this.buf = buf;
        this.pos = offset;
        this.count = Math.min(offset + length, buf.length);
        this.markedPos = offset;
    }

    //判断流是否打开
    private void ensureOpen() throws IOException {
        if (buf == null)
            throw new IOException("Stream closed");
    }

    //读取单个字符
    public int read() throws IOException {
		//进行 同步
        synchronized (lock) {
			//判断流是打开的
            ensureOpen();
            if (pos >= count)
                return -1;
            else
                return buf[pos++];
        }
    }

    //按照所传入的数组来进行读取，off为开始读取的位置，len为读取的长度
    public int read(char b[], int off, int len) throws IOException {
        synchronized (lock) {
            ensureOpen();
            if ((off < 0) || (off > b.length) || (len < 0) ||
                ((off + len) > b.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }

            if (pos >= count) {
                return -1;
            }

            int avail = count - pos;
            if (len > avail) {
                len = avail;
            }
            if (len <= 0) {
                return 0;
            }
			//// buf复制len个char到b
            System.arraycopy(buf, pos, b, off, len);
            pos += len;
            return len;
        }
    }

    //跳过字符。返回跳过的字符数。
    public long skip(long n) throws IOException {
        synchronized (lock) {
            ensureOpen();

            long avail = count - pos;
            if (n > avail) {
                n = avail;
            }
            if (n < 0) {
                return 0;
            }
            pos += n;
            return n;
        }
    }

    //告知该流是否已准备好读取
    public boolean ready() throws IOException {
        synchronized (lock) {
            ensureOpen();
            return (count - pos) > 0;
        }
    }

    //是否支持标记操作,true表示支持
    public boolean markSupported() {
        return true;
    }

    //标记流中的当前位置，与reset方法
    public void mark(int readAheadLimit) throws IOException {
        synchronized (lock) {
            ensureOpen();
            markedPos = pos;
        }
    }

    //将流重置为最近的标记位置，通常与mark方法共同使用
    public void reset() throws IOException {
        synchronized (lock) {
            ensureOpen();
            pos = markedPos;
        }
    }

    //关闭流并释放与之关联的任何系统资源
    public void close() {
        buf = null;
    }
}

```

简单实例

```java
    @Test
    public void test1() throws Exception{

        char [] datas={'a','b','c','d','e','f','g'};
        CharArrayReader charArrayReader=new CharArrayReader(datas,2,2);

        int len;
        //将数据源中的数据全部读出，并打印，此时pos应该与count想等
        while ((len = charArrayReader.read()) != -1) {
            System.out.print((char) len);
        }
    }
```

### CharArrayWriter

 字符数组输出流，与ByteArrayOutputStream类似，用于将字符写入到内置字符缓存数组char[] buf中、当此数组存放满员时会自动扩容。可使用 toCharArray() 和 toString() 获取数据、还可使用writeTo(Writer out)将buf写入到底层流中。

```java
package java.io;

import java.util.Arrays;

public
class CharArrayWriter extends Writer {
    //存储数据的字符缓冲数组
    protected char buf[];

    //字符缓冲数组的字符数
    protected int count;

    //构造方法
    public CharArrayWriter() {
        this(32);
    }

    //构造方法
    public CharArrayWriter(int initialSize) {
        if (initialSize < 0) {
            throw new IllegalArgumentException("Negative initial size: "
                                               + initialSize);
        }
        buf = new char[initialSize];
    }

    //写一个字符到缓冲数组中
    public void write(int c) {
		//进行同步操作
        synchronized (lock) {
			
            int newcount = count + 1;
            if (newcount > buf.length) {
				//进行数组扩容，并将原数组的数据复制到新的数组中
                buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
            }
            buf[count] = (char)c;
            count = newcount;
        }
    }

    //将字符写入缓冲区,off为开始写入的位置，len是写入的长度
    public void write(char c[], int off, int len) {
        if ((off < 0) || (off > c.length) || (len < 0) ||
            ((off + len) > c.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        synchronized (lock) {
            int newcount = count + len;
            if (newcount > buf.length) {
                buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
            }
            System.arraycopy(c, off, buf, count, len);
            count = newcount;
        }
    }

    //将字符串的一部分写入缓冲区,off为字符串的开始位置，len为开始位置向后的长度
    public void write(String str, int off, int len) {
        synchronized (lock) {
            int newcount = count + len;
            if (newcount > buf.length) {
                buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
            }
            str.getChars(off, off + len, buf, count);
            count = newcount;
        }
    }

    //将缓冲区的内容写入另一个字符流
    public void writeTo(Writer out) throws IOException {
        synchronized (lock) {
            out.write(buf, 0, count);
        }
    }

    //向写入器追加指定的字符序列
    public CharArrayWriter append(CharSequence csq) {
        String s = (csq == null ? "null" : csq.toString());
        write(s, 0, s.length());
        return this;
    }

    //向写入器追加指定的字符序列的子序列
    public CharArrayWriter append(CharSequence csq, int start, int end) {
        String s = (csq == null ? "null" : csq).subSequence(start, end).toString();
        write(s, 0, s.length());
        return this;
    }

    //向写入器追加单个字符
    public CharArrayWriter append(char c) {
        write(c);
        return this;
    }

    //重置缓冲区
    public void reset() {
        count = 0;
    }

    //返回输入数据的副本
    public char toCharArray()[] {
        synchronized (lock) {
            return Arrays.copyOf(buf, count);
        }
    }

    //返回字符缓冲区中字符的个数
    public int size() {
        return count;
    }

    //将输入数据转换为字符串
    public String toString() {
        synchronized (lock) {
            return new String(buf, 0, count);
        }
    }

    //刷新流
    public void flush() { }

    //关闭流
    public void close() { }

}

```

简单实例

```java
    @Test
    public void test3() throws Exception{

        CharArrayWriter charArrayWriter = new CharArrayWriter();
        charArrayWriter.write("Hello JAVA IO CharArrayWriter");
        FileWriter fileWriter = new FileWriter("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\test2.txt");

        charArrayWriter.writeTo(fileWriter);
        fileWriter.close();
        charArrayWriter.close();

    }
```

