Channel

Channel用于在字节缓冲区和位于通道另一侧的实体（通常是一个文件或套接字）之间有效地传输数据。

### channel继承体系

如图

![Channel](.\pic\Channel.png)

Channel:为所有通道的顶级接口，里面定义了该通道是否打开和关闭通道两个方法。
InterruptibleChannel是一个标记接口，当被通道使用时可以标示该通道是可以中断的（Interruptible）
AbstractInteruptibleChannel:一个可中断的通道的基础抽象实现类
SelectableChannel:通过选择器可以多路复用的通道
AbstractSelectableChannel：通过选择器可以多路复用的通道抽象实现类
ReadableByteChannel：一个可读的字节通道
WritableByteChannel：一个可写的字节通道
ByteChannel:将ReadableByteChannel和WritableByteChannel进行了整合，是一个可读可写的字节通道，即双向通道
SeekableByteChannel：维持当前postion的位置并且允许修改postion的字节通道
ScatteringByteChannel:一个把字节读到多个buffer的通道
GatheringByteChannel：一个从多个buffer中毒字节的通道
FileChannel：文件通道，从文件中读写数据；
DatagramChannel：通过 UDP 读写网络中数据；
SocketChannel：通过 TCP 读写网络中数据；
ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel

其中用到最多的是FileChannel,DatagramChannel,SocketChannel,ServerSocketChannel,FileChannel

### 文件通道

文件通道总是阻塞式的，因此不能被置于非阻塞模式。FileChannel对象不能直接创建。一个FileChannel实例只能通过在一个打开的file对（RandomAccessFile、FileInputStream或FileOutputStream）上调用getChannel( )方法获取。调用getChannel( )方法会返回一个连接到相同文件的FileChannel对象且该FileChannel对象具有与file对象相同的访问权

FileChannel源码

```java
package java.nio.channels;

import java.io.*;
import java.nio.ByteBuffer;
import java.nio.MappedByteBuffer;
import java.nio.channels.spi.AbstractInterruptibleChannel;
import java.nio.file.*;
import java.nio.file.attribute.FileAttribute;
import java.nio.file.spi.*;
import java.util.Set;
import java.util.HashSet;
import java.util.Collections;

//线程安全的用于读取、写入、映射和操作文件的通道。
public abstract class FileChannel
    extends AbstractInterruptibleChannel
    implements SeekableByteChannel, GatheringByteChannel, ScatteringByteChannel
{
    //构造方法
    protected FileChannel() { }

    //打开或创建一个文件，返回访问该文件的文件通道。
    public static FileChannel open(Path path,
                                   Set<? extends OpenOption> options,
                                   FileAttribute<?>... attrs)
        throws IOException
    {
        FileSystemProvider provider = path.getFileSystem().provider();
        return provider.newFileChannel(path, options, attrs);
    }

    @SuppressWarnings({"unchecked", "rawtypes"}) // generic array construction
    private static final FileAttribute<?>[] NO_ATTRIBUTES = new FileAttribute[0];

    //打开或创建一个文件，返回访问该文件的文件通道
    public static FileChannel open(Path path, OpenOption... options)
        throws IOException
    {
        Set<OpenOption> set = new HashSet<OpenOption>(options.length);
        Collections.addAll(set, options);
        return open(path, set, NO_ATTRIBUTES);
    }

    // -- Channel operations --

    //从这个通道读取字节序列到给定的缓冲区。
    public abstract int read(ByteBuffer dst) throws IOException;

   //从这个通道读取字节序列到给定缓冲区的子序列。
    public abstract long read(ByteBuffer[] dsts, int offset, int length)
        throws IOException;

    //从这个通道读取字节序列到给定的缓冲区
    public final long read(ByteBuffer[] dsts) throws IOException {
        return read(dsts, 0, dsts.length);
    }

    //将一个字节序列从给定的缓冲区写入此通道。
    public abstract int write(ByteBuffer src) throws IOException;

    //从给定缓冲区的子序列向该通道写入字节序列
    public abstract long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException;

    //从给定缓冲区向该通道写入字节序列
    public final long write(ByteBuffer[] srcs) throws IOException {
        return write(srcs, 0, srcs.length);
    }


    // -- Other operations --

    //返回通道的文件位置
    public abstract long position() throws IOException;

    //设置该通道的文件位置
    public abstract FileChannel position(long newPosition) throws IOException;

    //返回这个通道的文件的当前大小
    public abstract long size() throws IOException;

    //将此通道的文件截断为给定的大小
    public abstract FileChannel truncate(long size) throws IOException;

    //强制将此通道文件的任何更新写入包含它的存储设备
    public abstract void force(boolean metaData) throws IOException;

    //从这个通道的文件传输字节到给定的可写字节通道
    public abstract long transferTo(long position, long count,
                                    WritableByteChannel target)
        throws IOException;

    //将字节从给定的可读通道传输到此通道的文件中
    public abstract long transferFrom(ReadableByteChannel src,
                                      long position, long count)
        throws IOException;

    //从给定的文件位置开始，从这个通道读取字节序列到给定的缓冲区
    public abstract int read(ByteBuffer dst, long position) throws IOException;

    //从给定的文件位置开始，从给定的缓冲区向此通道写入字节序列
    public abstract int write(ByteBuffer src, long position) throws IOException;


    // -- Memory-mapped buffers --

    //用于文件映射模式的类型安全枚举
    public static class MapMode {

        //只读映射的模式
        public static final MapMode READ_ONLY
            = new MapMode("READ_ONLY");

        //读/写映射的模式
        public static final MapMode READ_WRITE
            = new MapMode("READ_WRITE");

        //私有(写时复制)映射的模式
        public static final MapMode PRIVATE
            = new MapMode("PRIVATE");

        private final String name;

        private MapMode(String name) {
            this.name = name;
        }
		
        public String toString() {
            return name;
        }

    }

   //将此通道文件的一个区域直接映射到内存中
    public abstract MappedByteBuffer map(MapMode mode,
                                         long position, long size)
        throws IOException;


    // -- Locks --

   //获取此通道文件的给定区域上的锁
    public abstract FileLock lock(long position, long size, boolean shared)
        throws IOException;

    //获取此通道文件的排他锁
    public final FileLock lock() throws IOException {
        return lock(0L, Long.MAX_VALUE, false);
    }

    //试图获取此通道文件的给定区域上的锁
    public abstract FileLock tryLock(long position, long size, boolean shared)
        throws IOException;

    //试图获取此通道文件的独占锁
    public final FileLock tryLock() throws IOException {
        return tryLock(0L, Long.MAX_VALUE, false);
    }

}

```

示例代码

```java
 @Test
    public void testFileReadNio() throws Exception {
        FileInputStream fileInputStream = new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\nio\\file\\FileDemo.java");
        FileChannel channel = fileInputStream.getChannel();


        ByteBuffer allocate = ByteBuffer.allocate(1024);


        while ( channel.read(allocate) != -1) {
            byte[] array = allocate.array();
            String str = new String(array);
            System.out.println(str);

            allocate.clear();
        }

        channel.close();
        fileInputStream.close();
    }


    @Test
    public void testFileWriteNio()throws Exception{

        FileOutputStream fileOutputStream=new FileOutputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\resources\\a.txt");
        FileChannel channel = fileOutputStream.getChannel();

        ByteBuffer buffer= Charset.forName("utf8").encode("hello java nio");
        while (channel.write(buffer) !=0){
            System.out.println(buffer.array().length);
        }

        channel.close();
        fileOutputStream.close();
    }



    @Test
    public void testFileReadWriteNio()throws Exception{


        FileInputStream fileInputStream = new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\nio\\file\\FileDemo.java");
        FileChannel readChannel = fileInputStream.getChannel();


        FileOutputStream fileOutputStream=new FileOutputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\resources\\a.txt");
        FileChannel writeChannel = fileOutputStream.getChannel();

        ByteBuffer allocate = ByteBuffer.allocate(1024);

        while (readChannel.read(allocate) !=-1){
            //进行翻转
            allocate.flip();
//            while (writeChannel.write(allocate) !=0){
//
//            }
            writeChannel.write(allocate);

            allocate.clear();
        }

        readChannel.close();
        writeChannel.close();
        fileInputStream.close();
        fileOutputStream.close();
    }
```

#### FileChannel内存映射

FileChannel类提供了一个名为map( )的方法，该方法可以在一个打开的文件和一个特殊类型的ByteBuffer之间建立一个虚拟内存映射。由map( )方法返回的MappedByteBuffer对象的行为在多数方面类似一个基于内存的缓冲区，只不过该对象的数据元素存储在磁盘上的一个文件中。调用get( )方法会从磁盘文件中获取数据，此数据反映该文件的当前内容，即使在映射建立之后文件已经被一个外部进程做了修改。通过文件映射看到的数据同您用常规方法读取文件看到的内容是完全一样的。相似地，对映射的缓冲区实现一个put( )会更新磁盘上的那个文件（假设对该文件您有写的权限），并且您做的修改对于该文件的其他阅读者也是可见的。

通过内存映射机制来访问一个文件会比使用常规方法读写高效得多，甚至比使用通道的效率都高。

示例代码

```java
    @Test
    public void testMapFile()throws Exception{


        FileInputStream fileInputStream = new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\java\\nio\\file\\FileDemo.java");
        FileChannel readChannel = fileInputStream.getChannel();

        FileOutputStream fileOutputStream=new FileOutputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\resources\\a.txt");
        FileChannel writeChannel = fileOutputStream.getChannel();

        MappedByteBuffer map = readChannel.map(FileChannel.MapMode.READ_ONLY, 100, 1000);

        writeChannel.write(map);
        readChannel.close();
        writeChannel.close();


      //  System.out.println(map);


    }
```

### socket通道

新的socket通道类可以运行非阻塞模式并且是可选择的。这两个性能可以激活大程序（如网络服务器和中间件组件）巨大的可伸缩性和灵活性。本节中我们会看到，再也没有为每个socket连接使用一个线程的必要了，也避免了管理大量线程所需的上下文交换总开销。借助新的NIO类，一个或几个线程就可以管理成百上千的活动socket连接了并且只有很少甚至可能没有性能损失。

#### ServerSocketChannel

ServerSocketChannel是一个基于通道的socket监听器。它同我们所熟悉的java.net.ServerSocket执行相同的基本任务，但是他能够在非阻塞模式下运行。

```java
package java.nio.channels;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.SocketOption;
import java.net.SocketAddress;
import java.nio.channels.spi.AbstractSelectableChannel;
import java.nio.channels.spi.SelectorProvider;

//面向流的监听套接字的可选通道
public abstract class ServerSocketChannel
    extends AbstractSelectableChannel
    implements NetworkChannel
{

    //构造方法，用来实例化ServerSocketChannel
    protected ServerSocketChannel(SelectorProvider provider) {
        super(provider);
    }

    //打开一个Server socket通道
    public static ServerSocketChannel open() throws IOException {
        return SelectorProvider.provider().openServerSocketChannel();
    }

    //返回一个操作集，标识此通道支持的操作
    public final int validOps() {
        return SelectionKey.OP_ACCEPT;
    }


    // -- ServerSocket-specific operations --

    //将通道的套接字绑定到本地地址，并将套接字配置为侦听连接。
    public final ServerSocketChannel bind(SocketAddress local)
        throws IOException
    {
        return bind(local, 0);
    }

    //将通道的套接字绑定到本地地址，并将套接字配置为侦听连接
    public abstract ServerSocketChannel bind(SocketAddress local, int backlog)
        throws IOException;

    public abstract <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
        throws IOException;

    //检索与此通道相关联的服务器套接字
    public abstract ServerSocket socket();

    //接受到此通道套接字的连接。
    public abstract SocketChannel accept() throws IOException;
	
    @Override
    public abstract SocketAddress getLocalAddress() throws IOException;

}
package java.nio.channels;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.SocketOption;
import java.net.SocketAddress;
import java.nio.channels.spi.AbstractSelectableChannel;
import java.nio.channels.spi.SelectorProvider;

//面向流的监听套接字的可选通道
public abstract class ServerSocketChannel
    extends AbstractSelectableChannel
    implements NetworkChannel
{

    //构造方法，用来实例化ServerSocketChannel
    protected ServerSocketChannel(SelectorProvider provider) {
        super(provider);
    }

    //打开一个Server socket通道
    public static ServerSocketChannel open() throws IOException {
        return SelectorProvider.provider().openServerSocketChannel();
    }

    //返回一个操作集，标识此通道支持的操作
    public final int validOps() {
        return SelectionKey.OP_ACCEPT;
    }


    // -- ServerSocket-specific operations --

    //将通道的套接字绑定到本地地址，并将套接字配置为侦听连接。
    public final ServerSocketChannel bind(SocketAddress local)
        throws IOException
    {
        return bind(local, 0);
    }

    //将通道的套接字绑定到本地地址，并将套接字配置为侦听连接
    public abstract ServerSocketChannel bind(SocketAddress local, int backlog)
        throws IOException;

    public abstract <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
        throws IOException;

    //检索与此通道相关联的服务器套接字
    public abstract ServerSocket socket();

    //接受到此通道套接字的连接。
    public abstract SocketChannel accept() throws IOException;
	
    @Override
    public abstract SocketAddress getLocalAddress() throws IOException;

}

```

示例代码

```java
    @Test
    public void testServerSocketChannel() throws Exception {

        ByteBuffer buffer = ByteBuffer.wrap("Hello I must be going.\r\n".getBytes(StandardCharsets.UTF_8));
        //打开ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        //绑定端口
        serverSocketChannel.socket().bind(new InetSocketAddress(9999));
        //设置为非阻塞模式
        serverSocketChannel.configureBlocking(false);


        while (true) {
            System.out.println("等待连接");
            //监听新进来的连接
            SocketChannel socketChannel =
                    serverSocketChannel.accept();

            if (socketChannel == null) {
                Thread.sleep(1000);
            } else {
                System.out.println("Incoming connection from: " + socketChannel.socket().getRemoteSocketAddress());
                buffer.rewind();
                socketChannel.write(buffer);
                socketChannel.close();
            }


        }
    }
```

#### SocketChannel

SocketChannel对应Socket，SocketChannel是一个连接到TCP网络套接字的通道。可以通过以下2种方式创建SocketChannel：

1. 打开一个SocketChannel并连接到互联网上的某台服务器。

2. 一个新连接到达ServerSocketChannel时，会创建一个SocketChannel。

   socketChannel源码

   ```java
   package java.nio.channels;
   
   import java.io.IOException;
   import java.net.Socket;
   import java.net.SocketOption;
   import java.net.SocketAddress;
   import java.nio.ByteBuffer;
   import java.nio.channels.spi.AbstractSelectableChannel;
   import java.nio.channels.spi.SelectorProvider;
   
   //面向流的连接套接字的可选通道
   public abstract class SocketChannel
       extends AbstractSelectableChannel
       implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel
   {
   
       //构造方法，用于实例化SocketChannel
       protected SocketChannel(SelectorProvider provider) {
           super(provider);
       }
   
       //打开一个socket通道
       public static SocketChannel open() throws IOException {
           return SelectorProvider.provider().openSocketChannel();
       }
   
       //打开一个套接字通道并将其连接到一个远程地址
       public static SocketChannel open(SocketAddress remote)
           throws IOException
       {
           SocketChannel sc = open();
           try {
               sc.connect(remote);
           } catch (Throwable x) {
               try {
                   sc.close();
               } catch (Throwable suppressed) {
                   x.addSuppressed(suppressed);
               }
               throw x;
           }
           assert sc.isConnected();
           return sc;
       }
   
       //返回一个操作集，标识此通道支持的操作
       public final int validOps() {
           return (SelectionKey.OP_READ
                   | SelectionKey.OP_WRITE
                   | SelectionKey.OP_CONNECT);
       }
   
   
       // -- Socket-specific operations --
       @Override
       public abstract SocketChannel bind(SocketAddress local)
           throws IOException;
   
       @Override
       public abstract <T> SocketChannel setOption(SocketOption<T> name, T value)
           throws IOException;
   
       //在不关闭通道的情况下关闭读取连接。
       public abstract SocketChannel shutdownInput() throws IOException;
   
       //不关闭通道的情况下关闭写入连接
       public abstract SocketChannel shutdownOutput() throws IOException;
   
       //检索与此通道关联的套接字
       public abstract Socket socket();
   
       //判断此通道的网络套接字是否已连接
       public abstract boolean isConnected();
   
       //明该通道上是否有正在进行的连接操作
       public abstract boolean isConnectionPending();
   
       //连接此通道的套接字。
       public abstract boolean connect(SocketAddress remote) throws IOException;
   
       //完成连接socket通道的过程
       public abstract boolean finishConnect() throws IOException;
   
       //返回此通道套接字连接到的远程地址
       public abstract SocketAddress getRemoteAddress() throws IOException;
   
       // -- ByteChannel operations --
   
       public abstract int read(ByteBuffer dst) throws IOException;
   
       public abstract long read(ByteBuffer[] dsts, int offset, int length)
           throws IOException;
   
       public final long read(ByteBuffer[] dsts) throws IOException {
           return read(dsts, 0, dsts.length);
       }
   
       public abstract int write(ByteBuffer src) throws IOException;
   
       public abstract long write(ByteBuffer[] srcs, int offset, int length)
           throws IOException;
   
       public final long write(ByteBuffer[] srcs) throws IOException {
           return write(srcs, 0, srcs.length);
       }
   
   
       @Override
       public abstract SocketAddress getLocalAddress() throws IOException;
   
   }
   
   ```

   示例代码，结合上面ServerSocketChannel一起使用

   ```java
       @Test
       public void ConnectAsync() throws Exception {
   
           InetSocketAddress addr = new InetSocketAddress("127.0.0.1", 9999);
           SocketChannel sc = SocketChannel.open();
           sc.configureBlocking(false);
           System.out.println("initiating connection");
           sc.connect(addr);
           while (!sc.finishConnect()) {
               System.out.println("doing something useless");
           }
           System.out.println("connection established"); // Do something with the connected socket // The SocketChannel is still nonblocking
           sc.close();
   
       }
   ```

   

#### DatagramChannel

DatagramChannel对应于 DatagramSocket，是一个能收发UDP包的通道。因为UDP是无连接的网络协议，所以不能像其它通道那样读取和写入。它发送和接收的是数据包。

示例代码

```java
    @Test
    public void testUDP() throws Exception {
        DatagramChannel channel = DatagramChannel.open();
        channel.socket().bind(new InetSocketAddress(9999));

        ByteBuffer buffer = ByteBuffer.allocate(48);

        byte b[];
        while (true) {
            buffer.clear();
            // 接受客户端发送数据
            SocketAddress socketAddress = channel.receive(buffer);

            if (socketAddress != null) {
                int position = buffer.position();
                b = new byte[position];
                buffer.flip();
                for (int i = 0; i < position; ++i) {
                    b[i] = buffer.get();
                }
                System.out.println("receive remote " + socketAddress.toString() + ":" + new String(b, "UTF-8"));
                //接收到消息后给发送方回应

                String message = "I has receive your message";
                ByteBuffer buffer1 = ByteBuffer.allocate(1024);
                buffer1.put(message.getBytes("UTF-8"));
                buffer1.flip();
                channel.send(buffer1, socketAddress);

            }
        }


    }
```

