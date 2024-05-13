StringReader&StringWriter

### StringReader

StringReader 是读, 从一个String中读取,所以需要一个String ,通过构造方法传递

```java
package java.io;



public class StringReader extends Reader {
	//内部字符串，通过构造方法传入
    private String str;
	//str字符串的长度
    private int length;
	//字符串下一个要读取的字符索引
    private int next = 0;
	//标记位置
    private int mark = 0;

    //构造方法
    public StringReader(String s) {
        this.str = s;
        this.length = s.length();
    }

    //确保流被打开没有关闭
    private void ensureOpen() throws IOException {
        if (str == null)
            throw new IOException("Stream closed");
    }

    //一次读取一个字符
    public int read() throws IOException {
        synchronized (lock) {
            ensureOpen();
            if (next >= length)
                return -1;
            return str.charAt(next++);
        }
    }

    //读取字符的一部分,最多为len字符，其中off为字符数组cbuf的下标的开始位置
    public int read(char cbuf[], int off, int len) throws IOException {
		//进行同步操作
        synchronized (lock) {
            ensureOpen();
            if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }
			//表示已经读取到了字符串的末尾
            if (next >= length)
                return -1;
			//表示读取的字符的个数为Math.min(length - next, len)
            int n = Math.min(length - next, len);
			//将str的字符复制到cbuf字符数组中
            str.getChars(next, next + n, cbuf, off);
            next += n;
            return n;
        }
    }

    //跳过流中的指定字符数。返回被跳过的字符数
    public long skip(long ns) throws IOException {
        synchronized (lock) {
            ensureOpen();
            if (next >= length)
                return 0;
            // Bound skip by beginning and end of the source
            long n = Math.min(length - next, ns);
            n = Math.max(-next, n);
            next += n;
            return n;
        }
    }

    //流是否已经准备好了
    public boolean ready() throws IOException {
        synchronized (lock) {
        ensureOpen();
        return true;
        }
    }

    //是否支持标记,true表示支持
    public boolean markSupported() {
        return true;
    }

    //进行标记
    public void mark(int readAheadLimit) throws IOException {
        if (readAheadLimit < 0){
            throw new IllegalArgumentException("Read-ahead limit < 0");
        }
        synchronized (lock) {
            ensureOpen();
            mark = next;
        }
    }

    //返回值标记的位置
    public void reset() throws IOException {
        synchronized (lock) {
            ensureOpen();
            next = mark;
        }
    }

    public void close() {
        str = null;
    }
}

```

简单实例

```java
    @Test
    public void test1() throws Exception{
        char cbuf[]=new char[5];
        String str="abcdefghijklmn";
        StringReader reader=new StringReader(str);
        int read = reader.read(cbuf, 2, 3);
        //此时cbuf数组中的值为{'\u0000','\u0000','a','b','c'}
        System.out.println(read);

        //此时读取的字符为d
        int read1 = reader.read();
        System.out.println((char) read1);


    }
```

### StringWriter

StringWriter是写, 写入到一个String中去,所以它内部提供了一个StringBuffer中用来保存数据

```java
package java.io;



public class StringWriter extends Writer {
	//用来对字符序列进行缓冲
    private StringBuffer buf;

    //构造方法
    public StringWriter() {
        buf = new StringBuffer();
        lock = buf;
    }

    //定义初始化缓冲区的大小
    public StringWriter(int initialSize) {
        if (initialSize < 0) {
            throw new IllegalArgumentException("Negative buffer size");
        }
        buf = new StringBuffer(initialSize);
        lock = buf;
    }

    //写入单个字符
    public void write(int c) {
        buf.append((char) c);
    }

    //写入字符数组的一部分，off为字符数组的开始位置，len为字符数组的长度
    public void write(char cbuf[], int off, int len) {
        if ((off < 0) || (off > cbuf.length) || (len < 0) ||
            ((off + len) > cbuf.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        buf.append(cbuf, off, len);
    }

    //写入一个字符串
    public void write(String str) {
        buf.append(str);
    }

    //写入一个字符串的一部分
    public void write(String str, int off, int len)  {
        buf.append(str.substring(off, off + len));
    }

    //追加一个字符序列
    public StringWriter append(CharSequence csq) {
        if (csq == null)
            write("null");
        else
            write(csq.toString());
        return this;
    }

    //追加一个字符序列的一部分
    public StringWriter append(CharSequence csq, int start, int end) {
        CharSequence cs = (csq == null ? "null" : csq);
        write(cs.subSequence(start, end).toString());
        return this;
    }

    //追加单个字符
    public StringWriter append(char c) {
        write(c);
        return this;
    }

    public String toString() {
        return buf.toString();
    }

    public StringBuffer getBuffer() {
        return buf;
    }

    public void flush() {
    }

    public void close() throws IOException {
    }

}

```

简单实例

```java
    @Test
    public void test2()throws Exception{

        StringWriter stringWriter=new StringWriter();
        stringWriter.write('a');
        System.out.println(stringWriter.getBuffer().toString());


        char [] c = {'b','c','d','e','f'};
        stringWriter.write(c,1,3);
        System.out.println(stringWriter.getBuffer().toString());

        stringWriter.write("hello java");
        System.out.println(stringWriter.getBuffer().toString());


    }
```

