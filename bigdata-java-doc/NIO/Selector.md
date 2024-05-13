Selector

### NIO选择器

在Java NIO中，选择器是可选择通道的多路复用器，可用作可以进入非阻塞模式的特殊类型的通道。它可以检查一个或多个NIO通道，并确定哪个通道准备好进行通信，即读取或写入。选择器用于使用单个线程处理多个通道。因此，它需要较少的线程来处理这些通道。

### Selector源码

```java
package java.nio.channels;

import java.io.Closeable;
import java.io.IOException;
import java.nio.channels.spi.SelectorProvider;
import java.util.Set;


//可选择通道对象的多路复用器，即选择器
public abstract class Selector implements Closeable {

    /**
     * Initializes a new instance of this class.
     */
    protected Selector() { }

    //打开一个选择器
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

    //判断此选择器是否为打开状态
    public abstract boolean isOpen();

    //
    public abstract SelectorProvider provider();

    //返回所有的SelectionKey的集合，代表注册在该Selector上的Channel
    public abstract Set<SelectionKey> keys();

    //返回此Selector已选择的SelectionKey的集合，
    public abstract Set<SelectionKey> selectedKeys();

    //执行一个立即返回的select()操作，该方法不会阻塞线程
    public abstract int selectNow() throws IOException;

    //可以设置超时时长的select()操作
    public abstract int select(long timeout)
        throws IOException;

    //监控所有注册Channel，当他们中间有需要处理的IO操作时，该方法返回，并将对应的SelectionKey加入被选择的SelectionKey集合中，该方法返回这些Channel的数量
    public abstract int select() throws IOException;

    //使一个还未返回的select()方法立即返回
    public abstract Selector wakeup();

    //关闭该选择器
    public abstract void close() throws IOException;

}

```

### SelectionKey源码

```java
package java.nio.channels;

import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;
import java.io.IOException;

//表示SelectableChannel和Selector之间的注册关系。每次向选择器注册通道时就会选择一个事件(选择键)。选择键包含两个表示为整数值的操作集。操作集的每一位都表示该键的通道所支持的一类可选择操作。
public abstract class SelectionKey {

    /**
     * Constructs an instance of this class.
     */
    protected SelectionKey() { }


    // -- Channel and selector operations --

    ////返回该创建该SelectionKey的SelectableChannel
    public abstract SelectableChannel channel();

    //返回该创建该SelectionKey的Selector
    public abstract Selector selector();

    //返回该key是否有效
    public abstract boolean isValid();

    
    public abstract void cancel();


    // -- Operation-set accessors --

    //获取感兴趣时间的集合
    public abstract int interestOps();

    
    public abstract SelectionKey interestOps(int ops);

    //获取通道已经准备就绪的操作的集合
    public abstract int readyOps();


    // -- Operation bits and bit-testing convenience methods --

    //读事件
    public static final int OP_READ = 1 << 0;

    //写事件
    public static final int OP_WRITE = 1 << 2;

    //连接事件
    public static final int OP_CONNECT = 1 << 3;

    //接受事件
    public static final int OP_ACCEPT = 1 << 4;

    //检测通道中读事件是否就绪
    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    //检测通道中写事件是否就绪
    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    //检测通道中连接事件是否就绪
    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    //检测通道中接受事件是否就绪
    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }


    // -- Attachments --

    private volatile Object attachment = null;

    private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
        attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
            SelectionKey.class, Object.class, "attachment"
        );

    
    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    
    public final Object attachment() {
        return attachment;
    }

}

```

选择器使用示例

```java
public class SelectSockets {
    public static int PORT_NUMBER = 1234;

    private ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) throws Exception {
        new SelectSockets().go(args);

    }

    private void go(String[] args) throws Exception {
        int port = PORT_NUMBER;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        }
        System.out.println("Listening on port " + port);


        // 1、创建Selector
        Selector selector = Selector.open();
        //2、创建ServerSocketChannel
        ServerSocketChannel serverChannel = ServerSocketChannel.open();

        ServerSocket serverSocket = serverChannel.socket();
        //3、绑定地址和端口
        serverSocket.bind(new InetSocketAddress(port));
        // 设置为非阻塞模式
        serverChannel.configureBlocking(false);
        //4、向Selector中注册Channel及感兴趣的事件，即监听事件，通过Selector监控serverChannel
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        //5、进行轮询查询就绪操作
        while (true) {
            //等待注册事件到达，否则一直阻塞
            int n = selector.select();
            if (n == 0) {
                continue;
            }
            //5、获取selector中选中项的迭代器，进行迭代
            Iterator it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();
                //针对感兴趣的事件进行处理，即针对OP_ACCEPT事件进行处理
                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel channel = server.accept();
                    registerChannel(selector, channel, SelectionKey.OP_READ);
                    sayHello(channel);
                }
                //针对感兴趣的事件进行处理，即针对OP_READ事件进行处理
                if (key.isReadable()) {
                    readDataFromSocket(key);
                }
                //事件处理完后将事件移除
                it.remove();
            }
        }


    }


    protected void registerChannel(Selector selector, SelectableChannel channel, int ops) throws Exception {
        if (channel == null) {
            return;
        }
        channel.configureBlocking(false);

        channel.register(selector, ops);
    }




    protected void readDataFromSocket(SelectionKey key) throws Exception {
        SocketChannel socketChannel = (SocketChannel) key.channel();
        int count;
        buffer.clear();
        while ((count = socketChannel.read(buffer)) > 0) {
            buffer.flip();

            while (buffer.hasRemaining()) {
                socketChannel.write(buffer);
            }

            buffer.clear();
        }
        if (count < 0) {
            socketChannel.close();
        }
    }


    private void sayHello(SocketChannel channel) throws Exception {
        buffer.clear();
        buffer.put("Hi there!\r\n".getBytes());
        buffer.flip();
        channel.write(buffer);
    }

}

```

