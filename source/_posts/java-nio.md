---
title: Java NIO解析
date: 2017-02-12 11:28:11
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

## 传统的BIO
一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。

## 伪异步I/O编程




## 常见Java IO模型

### 同步阻塞I/O(Blocking IO)




### 同步非阻塞I/O(Non-Blocking IO)
服务器实现模式为一个请求一个线程，但客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理


### 异步IO(Asynchronous IO)
服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。

## NIO模型I/O
Java的NIO是基于IO多路复用模型，其主要有三个核心的组件：Channel、Buffer和Selector

![](http://wx1.sinaimg.cn/mw690/78d85414ly1fociun9yiej20eu05ha9x.jpg "图1 nio核心组件")

在JAVA NIO中，到任何目的地(或来自任何地方)的所有数据都必须通过一个 Channel 对象。一个 Buffer 实质上是一个容器对象。发送给一个通道的所有对象都必须首先放到缓冲区中；同样地，从通道中读取的任何数据都要读到缓冲区中。

![](http://wx1.sinaimg.cn/mw690/78d85414ly1fociun9yiej20eu05ha9x.jpg "图2 nio读写示意图")

### Channel
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

### Buffer

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

### Selector

Selector是JAVA NIO中的多路复用器，配合SelectionKey使用，SelectionKey代表着一个Channel和Selector的关系的抽象，Channel向Selector注册的时候产生，由Selector维护。Selector维护着三个SelectionKey的集合：

* key set：这个集合包含所有向Selector注册的Channel产生的SelectionKey，这个集合中的SelectionKey是不能直接被修改的，除非SelectionKey被channel，并且发生select的时候SelectionKey才被移出。
* selected key set：这个集合是key set集合的子集，当有SelectionKey关联的Channel有Channel向Selector注册的IO事件就绪的时候并且有select操作，对应的SelectionKey会被放到selected key set中。因为这个集合中的SelectionKey可以通过直接调用Set的remove将SelectionKey移除。
* cancelled-key:这个集合是也是key set的子集。当有已经向Selector注册的Channel发生degistered的时候，SelectionKey将被放到这个集合，并且在下一次select的时候被从所有的集合中移出。

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1focjwn45gtj20ve0jodh2.jpg "图3 Selector的Selection Key集合流转图")

在开发过程中，我们可以将多个Channel注册到一个Selector实例中，用一个线程来处理所有的IO事件，我们也可以将多个Channel注册到多个Selector实例中，结合高效的线程模型可以达到很好的效果,
下面一个简单例子来说整个流转的过程。
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


## 参考文献

