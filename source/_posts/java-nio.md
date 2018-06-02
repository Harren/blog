---
title: Java NIO解析
date: 2018-03-12 11:28:11
categories:
- Java
tags:
- 高并发
---
Java NIO是NEW IO的简称，他是一种可以代替Java IO的一套新的IO机制，严格上来看，Java NIO与并发并无直接的关系，NIO 的创建目的是为了让 Java 程序员可以实现高速 I/O 而无需编写自定义的本机代码。NIO 将最耗时的 I/O 操作(即填充和提取缓冲区)转移回操作系统，因而可以极大地提高速度使用NIO可以大大提升线程的使用效率。

## 常见概念

### 同步与异步
同步和异步关注的是消息通信机制。所谓同步，就是在发出一个调用，在没有得到结果前，该调用就不返回，但是一旦调用返回，就得到返回值，由调用者主动等待调用的这个结果。

> 去银行办理业务，选择排队等候方式，这种就是同步等待消息通知，也就是需要我一直在等待银行办理业务情况。

异步则是相反，调用在发出后，这个调用就直接返回了，所以没有返回结果，换言之，当一个异步过程调用发生后，调用着不会立刻得到结果，而是在调用后，被调用者通过状态来通知调用者或者通过回调函数来处理这个调用。
> 去银行办理业务，从叫号机里面取出自己的好嘛，等到排到我者一号的时候由柜台的人通知我去办理业务，这种就是异步等待消息通知。在异步消息处理中，等待消息通知者往往注册一个回调机制，在所等待的事件被出发时由触发机制通过某种机制找到消息通知者。

### 阻塞与非阻塞

阻塞和非阻塞这两个概念与程序（线程）等待消息通知(无所谓同步或者异步)时的状态有关。也就是说阻塞与非阻塞主要是程序（线程）等待消息通知时的状态角度来说的。

> 排队等待或者叫号等待的过程中等待者除了等待消息通知之外不能做其他的事情，这种机制就是阻塞的。

阻塞调用是指调用结果返回之前，当前的线程会被挂起，调用线程只有在得到结果之后才会返回，非阻塞调用是指在不能立刻得到结果之前，该调用不会阻塞当前线程。

> 排队等待或者叫号等待的过程中等待者可以一边玩游戏一边听歌一边等待，这种机制就是非阻塞的。

## BIO模型
图1展示的是BIO的服务端的通信模型，采用BIO通信模型的服务端，通常由一个独立的Acceptor线程负责监听客户端的连接，它接收到客户端请求后为每个客户端创建一个新的线程进行链路处理，处理完成之后。通过输出流返回应答给客户端，线程销毁，这就是典型的一请求一应答通信模型。

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1fov09fcrgqj20ga05w0t6.jpg "图1 同步阻塞I/O服务端通信模型")

该架构最大的问题就是不具备弹性伸缩能力，当并发访问量增加后，服务端的线程个数和并发访问数呈现 1:1 的正比关系，由于线程是JAVA虚拟机非常宝贵的系统资源，当线程数膨胀之后，系统的性能急剧下降，随着并发量的继续增加，可能会发生句柄溢出、线程堆栈溢出等问题，并导致服务器最终宕机。

## 伪异步I/O模型
为了解决同步阻塞I/O面临的一个链路需要一个线程处理的问题，后端通过一个线程池来处理多个客户端的请求接入，通过线程池可以灵活的调配线程资源，设置线程的最大值，防止由于海量并发接入导致线程耗尽。当有新的客户端接入的时候，将客户端的 Socket 封装成一个 Task 投递到后端到线程池进行处理，JDK 的线程池维护一个消息队列和 N 个活跃线程堆消息队列重的任务进行处理，由于线程数可以设置消息队列的大小和最大线程数，因此，它的资源占用是可控的，无论多少个客户端并发访问，都不会导致资源的耗尽和宕机。

![](http://wx2.sinaimg.cn/mw690/78d85414ly1fov0ex0svgj20jx06g756.jpg "图2 伪异步I/O服务端通信模型")

伪异步I/O虽然避免了为每个请求都创建一个独立线程造成的资源耗尽问题，但是由于它底层的通信仍然采用同步阻塞模型，因此无法从根本上解决同步I/O导致的通信线程阻塞问题。

## NIO模型

Java的NIO是基于IO多路复用模型，其主要有三个核心的组件：Channel、Buffer和Selector

![](http://wx1.sinaimg.cn/mw690/78d85414ly1fociun9yiej20eu05ha9x.jpg "图3 nio核心组件")

在JAVA NIO中，到任何目的地(或来自任何地方)的所有数据都必须通过一个 Channel 对象。一个 Buffer 实质上是一个容器对象。发送给一个通道的所有对象都必须首先放到缓冲区中；同样地，从通道中读取的任何数据都要读到缓冲区中。

![](http://wx1.sinaimg.cn/mw690/78d85414ly1fociun9yiej20eu05ha9x.jpg "图4 nio读写示意图")

### 基本概念
#### Channel
通道是对原 I/O 包中的流的模拟，一个Channel可以和文件或者网络Socker对应，如果Channel对应这一个Socket，那么往这个Channel中写数据，就等同于向Socket写数据。Java中重要的通道实现主要包括一下几种

| 通道类型 | Description |
|--------|---------|
| FileChannel |从文件中读写数据|
| DatagramChannel |能通过UDP读写网络中的数据|
| SocketChannel | 能通过TCP读写网络中的数据。|
| ServerSocketChannel | 可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。|

下面使用 FileChannel 读取一个文件：
```Java
try(SeekableByteChannel seekableByteChannel = Files.newByteChannel(
                filename, EnumSet.of(StandardOpenOption.READ))) {
    seekableByteChannel.position(offset);
    ByteBuffer bb = ByteBuffer.allocate(maxLen);
    bb.clear();
    if(seekableByteChannel.read(bb) > 0) {
        return  bb.array();
    };
} catch (IOException e) {
    e.printStackTrace();
}
```

#### Buffer

Buffer 是一个对象，它包含一些要写入或者刚读出的数据，Java NIO中的Buffer用于和NIO通道进行交互。如你所知，数据是从通道读入缓冲区，从缓冲区写入到通道中的。缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。

在NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的。在写入数据时，它是写入到缓冲区中的。任何时候访问 NIO 中的数据，您都是将它放到缓冲区中。最常用的缓冲区类型是 ByteBuffer。一个 ByteBuffer 可以在其底层字节数组上进行 get/set 操作(即字节的获取和设置)。

使用Buffer读写数据一般遵循以下四个步骤：

1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者compact()方法

下面是一个使用Buffer的例子
```Java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();

//create buffer with capacity of 1024 bytes
ByteBuffer buf = ByteBuffer.allocate(1024);

int bytesRead = inChannel.read(buf); //read into buffer.
while (bytesRead != -1) {
  buf.flip();  //make buffer ready for read
  while(buf.hasRemaining()){
      System.out.print((char) buf.get()); // read 1 byte at a time
  }
  buf.clear(); //make buffer ready for writing
  bytesRead = inChannel.read(buf);
}
aFile.close();
```

#### Selector

Selector是JAVA NIO中的多路复用器，配合SelectionKey使用，SelectionKey代表着一个Channel和Selector的关系的抽象，Channel向Selector注册的时候产生，由Selector维护。Selector维护着三个SelectionKey的集合：

* key set：这个集合包含所有向Selector注册的Channel产生的SelectionKey，这个集合中的SelectionKey是不能直接被修改的，除非SelectionKey被channel，并且发生select的时候SelectionKey才被移出。
* selected key set：这个集合是key set集合的子集，当有SelectionKey关联的Channel有Channel向Selector注册的IO事件就绪的时候并且有select操作，对应的SelectionKey会被放到selected key set中。因为这个集合中的SelectionKey可以通过直接调用Set的remove将SelectionKey移除。
* cancelled-key:这个集合是也是key set的子集。当有已经向Selector注册的Channel发生registered的时候，SelectionKey将被放到这个集合，并且在下一次select的时候被从所有的集合中移出。

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1focjwn45gtj20ve0jodh2.jpg "图5 Selector的Selection Key集合流转图")

在开发过程中，我们可以将多个Channel注册到一个Selector实例中，用一个线程来处理所有的IO事件，我们也可以将多个Channel注册到多个Selector实例中，结合高效的线程模型可以达到很好的效果,

### NIO服务端序列图
![](http://wx2.sinaimg.cn/mw690/78d85414ly1fov3gc4uzej20g40ght9c.jpg "图6 NIO服务端创建序列图")

下面一个简单例子来说整个服务运行的过程。
```Java
public class NioEchoServer {

    private Selector selector;
    private ExecutorService tp = Executors.newCachedThreadPool();

    public static Map<Socket, Long> timeStat = new HashMap<>();

    private void startServer() throws Exception {

        selector = SelectorProvider.provider().openSelector();
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 设置SocketChannel为非阻塞模式
        serverSocketChannel.configureBlocking(false);

        InetSocketAddress isa = new InetSocketAddress(8000);
        serverSocketChannel.bind(isa);

        SelectionKey acceptKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        for(;;) {
            // select()方法是一个阻塞方法,如果没数据没,就等待,有数据就返回
            selector.select();

            Set<SelectionKey> readKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = readKeys.iterator();

            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();

                if(key.isAcceptable()) {
                    // 判断key所代表的channel是否处于Acceptable状态
                    doAccept(key);
                } else if(key.isValid() && key.isReadable()) {
                    // 判断key所代表的channel是否可以读取
                    doRead(key);
                } else if(key.isValid() && key.isWritable()) {
                    // 判断key所代表的channel是否可以写
                    doWrite(key);
                }
            }
        }
    }
}
```

### NIO客户端序列图
![](http://wx1.sinaimg.cn/mw690/78d85414ly1fov3gcnzs0j20g90i5751.jpg "图7 NIO客户端创建序列图")

可以根据一下的代码来理解整个流程
```java
public class NioEchoClient {

    private Selector selector;

    public void init(String ip, int port) throws IOException {

        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        selector = SelectorProvider.provider().openSelector();
        socketChannel.connect(new InetSocketAddress(ip, port));
        socketChannel.register(selector, SelectionKey.OP_CONNECT);
    }

    public void working() throws IOException {
        while (true) {
            if(!selector.isOpen()) {
                break;
            }
            selector.select();
            Iterator<SelectionKey> ite = selector.selectedKeys().iterator();
            while (ite.hasNext()) {
                SelectionKey key = ite.next();
                ite.remove();
                if(key.isConnectable()) {
                    connect(key);
                } else if(key.isReadable()) {
                    read(key);
                }
            }
        }
    }

    public void connect(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        // 如果正在连接,则完成连接
        if(channel.isConnectionPending()) {
            // 由于当前channel是非阻塞的,因此在connect方法返回时连接不一定建立了,后续需要再次确认
            channel.finishConnect();
        }

        channel.configureBlocking(false);
        channel.write(ByteBuffer.wrap(new String("hello server!\r\n").getBytes()));
        channel.register(selector, SelectionKey.OP_READ);
    }

    public void read(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();

        ByteBuffer buffer = ByteBuffer.allocate(100);
        channel.read(buffer);
        byte[] data = buffer.array();
        String msg = new String(data).trim();
        System.out.println("client receive msg:" + msg);
    }
}
```

## AIO模型

与NIO不同，当进行读写操作时，只须直接调用API的read或write方法即可。这两种方法均为异步的，对于读操作而言，当有流可读取时，操作系统会将可读的流传入read方法的缓冲区，并通知应用程序；对于写操作而言，当操作系统将write方法传递的流写入完毕时，操作系统主动通知应用程序。  即可以理解为，read/write方法都是异步的，完成后会主动调用回调函数。  在JDK1.7中，这部分内容被称作NIO.2，主要在java.nio.channels包下增加了下面四个异步通道：

* AsynchronousSocketChannel
* AsynchronousServerSocketChannel
* AsynchronousFileChannel
* AsynchronousDatagramChannel

其中的read/write方法，会返回一个带回调函数的对象，当执行完读取/写入操作后，直接调用回调函数。


## 参考文献
1. [Netty权威指南]()
2. [Java高并发程序设计]()

