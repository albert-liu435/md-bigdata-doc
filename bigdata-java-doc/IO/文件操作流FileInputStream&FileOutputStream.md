### 文件操作流FileInputStream&FileOutputStream

文件流包括字节流和字符流

字节流：FileInputStream、FileOutputStream

字符流：FileReader、FileWriter

### 文件操作字节流

FileInputStream

 FileInputStream流被称为文件字节输入流，意思指对文件数据以字节的形式进行读取操作如读取图片视频等

```java

package java.io;

import java.nio.channels.FileChannel;
import sun.nio.ch.FileChannelImpl;

public
class FileInputStream extends InputStream
{
    //文件描述符
    private final FileDescriptor fd;

    //文件路径
    private final String path;
	//JDK1.4新增的通道
    private FileChannel channel = null;
	//锁,用来进行同步操作
    private final Object closeLock = new Object();
	//用于判断流是否关闭
    private volatile boolean closed = false;

    //根据文件路径创建一个FileInputStream对象
    public FileInputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null);
    }

    //根据传入的文件创建FileInputStream对象
    public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);
		//获取当前应用的安全管理器SecruityManager，并对其是否有读取权限进行检测。
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);
        path = name;
        open(name);
    }

    //根据传入的文件描述符创建FileInputStream对象
    public FileInputStream(FileDescriptor fdObj) {
		//获取当前应用的安全管理器SecruityManager
        SecurityManager security = System.getSecurityManager();
        if (fdObj == null) {
            throw new NullPointerException();
        }
        if (security != null) {
            security.checkRead(fdObj);
        }
        fd = fdObj;
        path = null;
        fd.attach(this);
    }

    //调用java native方法打开文件
    private native void open0(String name) throws FileNotFoundException;

    //打开文件
    private void open(String name) throws FileNotFoundException {
        open0(name);
    }

    //读取文件
    public int read() throws IOException {
        return read0();
    }
	//调用native方法read0()来实际读取文件
    private native int read0() throws IOException;

    //调用native方法从off位置读取len长度的字节到b字节数组中
    private native int readBytes(byte b[], int off, int len) throws IOException;

    //从文件中读取数据到b字节数组中
    public int read(byte b[]) throws IOException {
        return readBytes(b, 0, b.length);
    }

    //从off位置读取len长度的字节到b字节数组中
    public int read(byte b[], int off, int len) throws IOException {
        return readBytes(b, off, len);
    }

    //跳过n字节的数组
    public long skip(long n) throws IOException {
        return skip0(n);
    }
	//调用native方法跳过n字节的数组
    private native long skip0(long n) throws IOException;

    //返回可以从该输入流读取(或跳过)的剩余字节数
    public int available() throws IOException {
        return available0();
    }
	//调用native方法，并返回可以从该输入流读取(或跳过)的剩余字节数
    private native int available0() throws IOException;

    //关闭流
    public void close() throws IOException {
        synchronized (closeLock) {
            if (closed) {
                return;
            }
            closed = true;
        }
        if (channel != null) {
           channel.close();
        }

        fd.closeAll(new Closeable() {
            public void close() throws IOException {
               close0();
           }
        });
    }

    //获取文件描述符
    public final FileDescriptor getFD() throws IOException {
        if (fd != null) {
            return fd;
        }
        throw new IOException();
    }

    //jdk1.4新增，获取FileChannel
    public FileChannel getChannel() {
        synchronized (this) {
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, true, false, this);
            }
            return channel;
        }
    }
	// 用于设置fd的内存地址偏移量，牵扯到JNI编程
    private static native void initIDs();
	//调用本地方法进行资源关闭
    private native void close0() throws IOException;

    static {
        initIDs();
    }

    //重写finalize方法用于确认资源关闭
    protected void finalize() throws IOException {
        if ((fd != null) &&  (fd != FileDescriptor.in)) {
            close();
        }
    }
}

```

简单实例

```java
    @Test
    public void test9() throws Exception {
        byte[] buffer = new byte[2];

        FileInputStream fileInputStream = new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\FileIODemo.java");
        int length = 0;


        while ((length =fileInputStream.read(buffer) )!= -1) {
            //System.out.println(new String(buffer,0,length));
            System.out.print(new String(buffer,0,length));
            System.out.flush();

        }


        //        FileOutputStream fileOutputStream=new FileOutputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\ds.java");
//        while (fileInputStream.read(buffer) !=-1){
//            fileOutputStream.write(buffer);
//        }

        System.out.println("操作完成");

    }
```

FileOutputStream

 FileOutputStream流是指文件字节输出流，专用于输出原始字节流如图像数据等，其继承OutputStream类，拥有输出流的基本特性

```java

package java.io;

import java.nio.channels.FileChannel;
import sun.nio.ch.FileChannelImpl;
public
class FileOutputStream extends OutputStream
{
    //文件描述符
    private final FileDescriptor fd;

    //是否追加，当未true时则向文件中追加内容
    private final boolean append;

    //JDK1.4新增的通道
    private FileChannel channel;

    //文件路径
    private final String path;
	//锁
    private final Object closeLock = new Object();
	//用于判断流是否关闭
    private volatile boolean closed = false;

    //根据文件路径创建FileOutputStream,默认对文件不进行追加
    public FileOutputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null, false);
    }

  //根据文件路径和加创建FileOutputStream，append为true表示对文件进行追加
    public FileOutputStream(String name, boolean append)
        throws FileNotFoundException
    {
        this(name != null ? new File(name) : null, append);
    }

    //根据文件对象创建FileOutputStream，默认对文件不进行追加
    public FileOutputStream(File file) throws FileNotFoundException {
        this(file, false);
    }

    ////根据文件对象创建FileOutputStream，append为true表示对文件进行追加
    public FileOutputStream(File file, boolean append)
        throws FileNotFoundException
    {
        String name = (file != null ? file.getPath() : null);
		//获取java安全管理器，对当前是否具有写入的权限进行检查。
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkWrite(name);
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        this.fd = new FileDescriptor();
        fd.attach(this);
        this.append = append;
        this.path = name;

        open(name, append);
    }

    //根据文件描述符创建FileOutputStream
    public FileOutputStream(FileDescriptor fdObj) {
        SecurityManager security = System.getSecurityManager();
        if (fdObj == null) {
            throw new NullPointerException();
        }
        if (security != null) {
            security.checkWrite(fdObj);
        }
        this.fd = fdObj;
        this.append = false;
        this.path = null;

        fd.attach(this);
    }

    //调用native方法打开文件，append为ture表示对文件进行追加，false表示对文件进行覆盖
    private native void open0(String name, boolean append)
        throws FileNotFoundException;

    //打开文件，append为ture表示对文件进行追加，false表示对文件进行覆盖
    private void open(String name, boolean append)
        throws FileNotFoundException {
        open0(name, append);
    }

    //调用native方法对文件进行数据写入，append为ture表示对文件进行追加
    private native void write(int b, boolean append) throws IOException;

    //
    public void write(int b) throws IOException {
        write(b, append);
    }

    //
    private native void writeBytes(byte b[], int off, int len, boolean append)
        throws IOException;

    //
    public void write(byte b[]) throws IOException {
        writeBytes(b, 0, b.length, append);
    }

    //
    public void write(byte b[], int off, int len) throws IOException {
        writeBytes(b, off, len, append);
    }

    //关闭流及关联的系统资源
    public void close() throws IOException {
        synchronized (closeLock) {
            if (closed) {
                return;
            }
            closed = true;
        }

        if (channel != null) {
            channel.close();
        }

        fd.closeAll(new Closeable() {
            public void close() throws IOException {
               close0();
           }
        });
    }

    //获取文件描述符
     public final FileDescriptor getFD()  throws IOException {
        if (fd != null) {
            return fd;
        }
        throw new IOException();
     }

    //jdk1.4新增，获取FileChannel
    public FileChannel getChannel() {
        synchronized (this) {
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, false, true, append, this);
            }
            return channel;
        }
    }

    //重写了finalize方法，调用close方法保证流与其关联的资源能够获得释放。如果是标准输出流或者错误输出流，还会调用flush方法
    protected void finalize() throws IOException {
        if (fd != null) {
            if (fd == FileDescriptor.out || fd == FileDescriptor.err) {
                flush();
            } else {
                /* if fd is shared, the references in FileDescriptor
                 * will ensure that finalizer is only called when
                 * safe to do so. All references using the fd have
                 * become unreachable. We can call close()
                 */
                close();
            }
        }
    }

    private native void close0() throws IOException;

    private static native void initIDs();

    static {
        initIDs();
    }

}

```

简单示例

```java
    @Test
    public void test10()throws Exception{
        byte[] buffer=new byte[1024];

        FileInputStream fileInputStream=new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\FileIODemo.java");
        FileOutputStream fileOutputStream=new FileOutputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\io\\ds.java");
        while (fileInputStream.read(buffer) !=-1){
            fileOutputStream.write(buffer);
        }

        System.out.println("操作完成");

    }
```

