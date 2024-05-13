### SequenceInputStream

SequenceInputStream是一个合并流，当我们从多个流中读取数据，并希望他们合并成一个流作为数据源时，我们就可以使用SequenceInputStream.

```java
package java.io;

import java.io.InputStream;
import java.util.Enumeration;
import java.util.Vector;

//SequenceInputStream是一个合并流,用于将多个输入流合并为一输入流，内部采用枚举来实现多个流的合并
public
class SequenceInputStream extends InputStream {
	//枚举，用来存放多个不同的流，并不断的调用nextElement方法对多个流进行合并
    Enumeration<? extends InputStream> e;
	//当前读取的流
    InputStream in;

    //通过传入的Enumeration来创建SequenceInputStream
    public SequenceInputStream(Enumeration<? extends InputStream> e) {
        this.e = e;
        try {
            //继续在下一个流中读取
            nextStream();
        } catch (IOException ex) {
            // This should never happen
            throw new Error("panic");
        }
    }

    //通过两个流来创建SequenceInputStream
    public SequenceInputStream(InputStream s1, InputStream s2) {
        Vector<InputStream> v = new Vector<>(2);

        v.addElement(s1);
        v.addElement(s2);
        e = v.elements();
        try {
            nextStream();
        } catch (IOException ex) {
            // This should never happen
            throw new Error("panic");
        }
    }

    //继续在下一个流中读取
    final void nextStream() throws IOException {
        if (in != null) {
            in.close();
        }

        if (e.hasMoreElements()) {
            in = (InputStream) e.nextElement();
            if (in == null)
                throw new NullPointerException();
        }
        else in = null;

    }

    //
    public int available() throws IOException {
        if (in == null) {
            return 0; // no way to signal EOF from available()
        }
        return in.available();
    }

    //从流中读取一个字节的数据
    public int read() throws IOException {
        while (in != null) {
            //从流中读取一个字节
            int c = in.read();
            //不为-1表示没有到达结尾
            if (c != -1) {
                return c;
            }
            nextStream();
        }
        return -1;
    }

    //读取流中的数据到b数组中
    public int read(byte b[], int off, int len) throws IOException {
        //判断流是否为null
        if (in == null) {
            return -1;
            //判断字节数组是否为null
        } else if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }
        do {
            //从流中进行读取字节到字节数组b中
            int n = in.read(b, off, len);
            if (n > 0) {
                return n;
            }
            nextStream();
        } while (in != null);
        return -1;
    }

    //关闭该输入流并释放与该流关联的任何系统资源
    public void close() throws IOException {
        do {
            nextStream();
        } while (in != null);
    }
}

```

简单实例

```java
    @Test
    public void test2() throws Exception{
        //Enumeration<? extends InputStream> e;
        StringBuffer str = new StringBuffer();
        Vector<InputStream> vector=new Vector<>();
        vector.add(new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\ByteArrayDemo.java"));
        vector.add(new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\FileIODemo.java"));
        Enumeration<InputStream> enumeration=vector.elements();

        SequenceInputStream sequenceInputStream=new SequenceInputStream(enumeration);
        int len;
        while((len = sequenceInputStream.read()) != -1){
            str.append((char)len);
        }

        sequenceInputStream.close();
        System.out.println(str);




    }
```

