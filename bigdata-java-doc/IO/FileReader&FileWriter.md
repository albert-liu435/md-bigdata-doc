FileReader&FileWriter

FileReader是从文件中读取字符流的类，继承InputStreamReader，所以我们先看一下InputStreamReader的源码

### InputStreamReader

InputStreamReader是从字节流到字符流的桥：它读取字节，并使用指定的charset将其解码为字符 。 它使用的字符集可以由名称指定，也可以被明确指定，或者可以接受平台的默认字符集。每个调用InputStreamReader的read（）方法之一可能会导致从底层字节输入流读取一个或多个字节。 为了使字节有效地转换为字符，可以从底层流读取比满足当前读取操作所需的更多字节。

```java
package java.io;

import java.nio.charset.Charset;
import java.nio.charset.CharsetDecoder;
import sun.nio.cs.StreamDecoder;

public class InputStreamReader extends Reader {

    private final StreamDecoder sd;

    //创建一个使用默认字符集的InputStreamReader
    public InputStreamReader(InputStream in) {
        super(in);
        try {
            sd = StreamDecoder.forInputStreamReader(in, this, (String)null); // ## check lock object
        } catch (UnsupportedEncodingException e) {
            // The default encoding should always be available
            throw new Error(e);
        }
    }

    //创建一个使用给定字符集的InputStreamReader
    public InputStreamReader(InputStream in, String charsetName)
        throws UnsupportedEncodingException
    {
        super(in);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
    }

    //创建一个使用给定字符集解码器的InputStreamReader
    public InputStreamReader(InputStream in, Charset cs) {
        super(in);
        if (cs == null)
            throw new NullPointerException("charset");
        sd = StreamDecoder.forInputStreamReader(in, this, cs);
    }

    //创建一个使用命名字符集的InputStreamReader
    public InputStreamReader(InputStream in, CharsetDecoder dec) {
        super(in);
        if (dec == null)
            throw new NullPointerException("charset decoder");
        sd = StreamDecoder.forInputStreamReader(in, this, dec);
    }

    //返回此流使用的字符编码的名称
    public String getEncoding() {
        return sd.getEncoding();
    }

    //读一个字符
    public int read() throws IOException {
        return sd.read();
    }

    //将字符读入数组的一部分
    public int read(char cbuf[], int offset, int length) throws IOException {
        return sd.read(cbuf, offset, length);
    }

    //告诉这个流是否准备好被读取
    public boolean ready() throws IOException {
        return sd.ready();
    }
	//关闭流并释放与之相关联的任何系统资源
    public void close() throws IOException {
        sd.close();
    }
}

```

### FileReader

```java
package java.io;



public class FileReader extends InputStreamReader {

   //创建一个新的 FileReader ，给定要读取的文件的名称
    public FileReader(String fileName) throws FileNotFoundException {
        super(new FileInputStream(fileName));
    }

   //创建一个新的 FileReader ，给出 File读取
    public FileReader(File file) throws FileNotFoundException {
        super(new FileInputStream(file));
    }

   //创建一个新的 FileReader ，给定 FileDescriptor读取
    public FileReader(FileDescriptor fd) {
        super(new FileInputStream(fd));
    }

}

```

### OutputStreamWriter

OutputStreamWriter是字符的桥梁流以字节流：向其写入的字符编码成使用指定的字节charset 。 它使用的字符集可以由名称指定，也可以被明确指定，或者可以接受平台的默认字符集。
每次调用write（）方法都会使编码转换器在给定字符上被调用。 所得到的字节在写入底层输出流之前累积在缓冲区中。 可以指定此缓冲区的大小，但是默认情况下它大部分用于大多数目的。 请注意，传递给write（）方法的字符不会缓冲。

```java
package java.io;

import java.nio.charset.Charset;
import java.nio.charset.CharsetEncoder;
import sun.nio.cs.StreamEncoder;

public class OutputStreamWriter extends Writer {

    private final StreamEncoder se;

    //创建一个使用命名字符集的OutputStreamWriter。
    public OutputStreamWriter(OutputStream out, String charsetName)
        throws UnsupportedEncodingException
    {
        super(out);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        se = StreamEncoder.forOutputStreamWriter(out, this, charsetName);
    }

    //创建一个使用默认字符编码的OutputStreamWriter
    public OutputStreamWriter(OutputStream out) {
        super(out);
        try {
            se = StreamEncoder.forOutputStreamWriter(out, this, (String)null);
        } catch (UnsupportedEncodingException e) {
            throw new Error(e);
        }
    }

    //创建一个使用给定字符集的OutputStreamWriter。
    public OutputStreamWriter(OutputStream out, Charset cs) {
        super(out);
        if (cs == null)
            throw new NullPointerException("charset");
        se = StreamEncoder.forOutputStreamWriter(out, this, cs);
    }

    //创建一个使用给定字符集编码器的OutputStreamWriter。
    public OutputStreamWriter(OutputStream out, CharsetEncoder enc) {
        super(out);
        if (enc == null)
            throw new NullPointerException("charset encoder");
        se = StreamEncoder.forOutputStreamWriter(out, this, enc);
    }

    //返回此流使用的字符编码的名称。
    public String getEncoding() {
        return se.getEncoding();
    }

    //
    void flushBuffer() throws IOException {
        se.flushBuffer();
    }

    //写一个字符
    public void write(int c) throws IOException {
        se.write(c);
    }

    //写入字符数组的一部分
    public void write(char cbuf[], int off, int len) throws IOException {
        se.write(cbuf, off, len);
    }

    //写一个字符串的一部分
    public void write(String str, int off, int len) throws IOException {
        se.write(str, off, len);
    }

    //刷新流
    public void flush() throws IOException {
        se.flush();
    }
	//关闭流，先刷新。 一旦流已关闭，进一步的write（）或flush（）调用将导致抛出IOException。 关闭以前关闭的流无效
    public void close() throws IOException {
        se.close();
    }
}

```

### FileWriter

```java

package java.io;



public class FileWriter extends OutputStreamWriter {

    //构造一个给定文件名的FileWriter对象
    public FileWriter(String fileName) throws IOException {
        super(new FileOutputStream(fileName));
    }

    //给一个File对象构造一个FileWriter对象。
    public FileWriter(String fileName, boolean append) throws IOException {
        super(new FileOutputStream(fileName, append));
    }

    //给一个File对象构造一个FileWriter对象。
    public FileWriter(File file) throws IOException {
        super(new FileOutputStream(file));
    }

    //构造一个FileWriter对象，给出一个带有布尔值的文件名，表示是否附加写入的数据
    public FileWriter(File file, boolean append) throws IOException {
        super(new FileOutputStream(file, append));
    }

    //构造与文件描述符关联的FileWriter对象。
    public FileWriter(FileDescriptor fd) {
        super(new FileOutputStream(fd));
    }

}

```

简单实例

```java
    @Test
    public void test1() throws Exception {

        char[] buffer = new char[2048];

        FileReader fr = new FileReader(new File("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\test1.txt"));

        FileWriter fw = new FileWriter(new File(
                "C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\test3.txt"));
        int len;
        while ((len = fr.read(buffer)) != -1) {
            fw.write(buffer, 0, len);
        }
        fw.flush();

        fw.close();
        fr.close();

        System.out.println("复制完成");
    }
```

