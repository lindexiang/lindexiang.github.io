---
title: socket原理
images: /images/摘要配图/
date: 2018-10-21 23:41:33
tags:
    - 计算机网络
    - socket编程
    - TCP/IP
categories: socket编程
image: http://pbhb4py13.bkt.clouddn.com/2018-10-21-thanh-nguyen-duy-1111692-unsplash.jpg
top : 100

---
# socket编程介绍
## 简介
进程间通信的方法有管道，命名管道，信号，消息队列，共享内存，信号量，这些方法要求通信的两个线程位于同一个主机。对于非同一个主机可以使用TCP／IP协议搞定。
套接字是网络通信的基础。当一个程序创建了一个socket并与端口号绑定时候，意味着向TCP/IP协议申明了对端口的占有。比如端口为8080。所有目标为8080端口的TCP/IP协议包都会被该程序收到。Accept函数，就是抽象的TCP连接建立过程，accept函数返回的是新socket是指代本次连接。一个链接包括两个部分，一个是源IP和源端口，另一部分是目标IP和目标端口。所以accept可以产生多个socket。   
**Socket类似于是对TCP/IP协议协议的抽象，对外提供了编程的接口、使用这个接口可以方便的使用tcp/IP的功能。所有socket是对TCP/IP协议栈的抽象，而不是简单的映射关系。**
<!-- more -->
![9B702DEB-FAF5-4F31-9630-5E634D150B3B](http://pbhb4py13.bkt.clouddn.com/2018-10-21-9B702DEB-FAF5-4F31-9630-5E634D150B3B.png)
![0E6B62B2-5328-44E8-A2E1-873F6558B138](http://pbhb4py13.bkt.clouddn.com/2018-10-21-0E6B62B2-5328-44E8-A2E1-873F6558B138.png)
socket包括客户端和服务端。
Socket是客户端使用的，构造函数如下

```java
Socket(String host, int port, InetAddress localAddress, int localPort)throws IOException
```
会尝试和服务端建立链接，失败抛出异常，成功后返回socket对象，是一个同步阻塞IO
serverSocket是服务端使用的，构造函数如下

```java
ServerSocket(int port, int backlog, InetAddress bindAddr)throws IOException
```
其中 port是服务端监听的端口号，backlog是accept队列的大小。bindAddr是服务端绑定的IP

## listen，connect，accept的流程以及原理
客户端发起socket链接后，会使用TCP的3次握手和服务端建立链接，而serverSocket使用accept函数来得到链接的客户端。
流程图如下
![673B10A2-B926-4778-8908-FE5167CD6B3C](http://pbhb4py13.bkt.clouddn.com/2018-10-21-673B10A2-B926-4778-8908-FE5167CD6B3C.png)

1. 服务端new一个serverSocket后，会绑定一个端口设置状态为LISTEN，监听端口的情况。同时内核生成2个队列，一个是syn队列。另一个是accept队列。其中ACCEPT队列的长度由backlog决定。
2. 服务端调用accept后，将会阻塞，等待ACCEPT队列中有元素。
3. 客户端调用connect函数后，将发起SYN请求，请求与服务端建立TCP连接，第一次握手
4. 服务端收到SYN请求后，将client放入syn队列中，并给客户返回ACK，此时还会携带一个要和客户端建立连接的请求标志SYN，称为第二次握手。
5. 客户端收到SYN+ACK后，connect函数返回，并发送ACK给服务器端，称为第三次握手。
6. 服务端收到ACK后，从SYN队列中移除client，并加入ACCEPT队列中，而accept函数等到资源，阻塞中唤醒，从ACCEPT队列中取出请求方，重新建立一个连接socketId，返回。

## BIO编程
网络编程的基本模型是C/S，即两个进程间的通信。
服务端提供了IP和监听接口，客户端如果想对监听的地址发起连接请求,通过三次握手连接，如果连接建立成功，双方就可以通过socket通信。
传统的同步阻塞模型开发中，ServerSocket负责绑定IP地址,启动监听端口；socket负责发起连接操作。连接成功后，双方通过输入流和输出流进行同步阻塞通信。
BIO通信模型： 采用BIO通信模型的服务端，通常有一个独立的accept线程负责监听客户端的连接，他收到客户端连接请求后为每个客户端创建一个新的线程进行数据处理。处理完成后通过输入流返回给应答客户端。线程销毁。
![BC7DF720-2072-435A-B68D-C2A4836A6C28](http://pbhb4py13.bkt.clouddn.com/2018-10-21-BC7DF720-2072-435A-B68D-C2A4836A6C28.png)
BIO的问题是当有一个新的客户端请求接入时候，服务端要建立一个新的线程来处理这个链路。该模型缺乏伸缩能力，当客户端并发量增加后，由于服务端和客户端的访问数量是1:1，则在线程数增加后，系统的性能就会下降很多。

## BIO使用线程池
我们可以使用线程池来实现一个线程或者多个线程来处理N个客户端模型。底层还是使用同步阻塞IO，称为伪异步IO模型
![0D5A9C5B-FE0C-4389-8D9C-6C02985252DF](http://pbhb4py13.bkt.clouddn.com/2018-10-22-0D5A9C5B-FE0C-4389-8D9C-6C02985252DF.png)

```java
public static void start(int port) throws IOException {
        if (server != null) return;
        try {
            server = new ServerSocket(port);
            System.out.println("服务器已启动，端口号: " + port);
            //无限循环监听客户端连接
            while (true) {
                Socket socket = server.accept();
                executorService.execute(new ServerHandler(socket));
           //new Thread(new ServerHandler(socket)).start();
            }
        } finally {
            if (server != null) {
                System.out.println("服务器已关闭");
                server.close();
                server = null;
            }
        }
    }
}
```
从上述的一部分代码中看出，使用线程池来管理了socket。使用了线程池后，不仅能帮我们管理线程,使用FixedThreadPool能有效控制线程的最大数量，保证系统的资源的控制。
但是，限制了线程的最大数量，如果发生大量的并发请求，超过最大数量的线程就只能等待，直到线程池中有空闲的线程可以被复用。而对socket的输入流读取时会一直阻塞，直到发生：

1. 有数据可以读取
2. 可用数据读取完毕
3. 发生空指针异常
       
所以在读取数据较慢时候(数据量大、网络传输慢)，其他的消息只能等待。
## NIO编程
从上述的BIO模型中，可以很明显的看出其实大部分的线程都是在等待数据传输过来，这部分的线程一直都是在等待的状态，相当于在IO模型的第一部分wait data的状态。使用NIO，在第一阶段不会创建线程，而是采用channel来等待数据，当数据读取完毕后才创建线程来处理。
NIO提供了与传统的BIO模型中的socket和ServerSocket相对应的socketChannel和ServerSocketChannel两种不同的套接字实现。
![3CE21703-CC96-4CDC-A974-4E172B3A92CC](http://pbhb4py13.bkt.clouddn.com/2018-10-22-3CE21703-CC96-4CDC-A974-4E172B3A92CC.png)
### channel理解
Channel,中文意思“通道”，表示IO源与目标打开的连接，类似于传统的“流”。但是Channel不能直接访问数据，需要和缓冲区buffer进行交互。**打个比方，山西有煤，我们想要，于是乎建了一条铁路连通到山西，这条铁路就是这里的"Channel"，而buffer类似于火车。**
使用channel来处理IO请求的工作原理如下。buffer是内核地址空间的，也可以把channel理解为一条TCP通道。
![](http://pbhb4py13.bkt.clouddn.com/2018-10-22-15401396825782.jpg)

> 从上面的图中可以看出可以设置一个API直接从内核地址空间的buffer中读取数据，不要将数据copy到用户空间中。这样可以极大的提高效率。
程序的数据要先到jvm用户地址缓冲区，再到计算机内存中。也就是不能直接和计算机内存进行数据操作。那么通过内存映射文件，也就是上诉方式，内部以直接缓冲区和内存映射文件实现了直接访问内存。这样大大提高了效率。但是也有缺点，如果写过去的数据很大，然后jvm没有及时回收内存，那么结果可想而知。因此，这个适用于那种长久存放在内存不也不会被短时间内就需要回收的数据。
### 缓冲区buffer
Buffer是一个数据的缓冲对象。在NIO中，所有的数据都是通过buffer处理。channel的数据要先读取到buffer中才可以直接使用。directBuffer指的是内核上的缓冲对象。**netty是直接读取这部分的数据，少了将数据从内核拷贝到用户的过程。**
### select选择器
> Selector(选择器)是java中检测多个NIO通道，并能够知晓通道是否有读写事件的组件，这样子一个线程能管理多个channel，从而管理多个网络。
     
使用一个Selector来管理多个channel的好处是只要很少的线程来处理通道。对于操作系统来说，线程之间的上下文切换需要很大的开销，而且每个线程都占用系统的资源(内存)。
当通道触发了一个事件，表明这个事件已经就绪。所以某个客户端的channel成功连接上服务器称为**连接就绪**。一个`serverSocketChannel`准备好接收新进入的连接称为**接受就绪**。一个有数据可读的通道为**读就绪**。等待写数据的通道为**写就绪**。

> 使用NIO编程相当于**多路IO复用模型**的第一部分。在数据准备阶段由Selector来搞定。当数据传输完毕，Selector会调用用户线程来进行真正的IO读写。这是第二部分的内容。则NIO是一个同步非阻塞IO模型。同步和异步指的是再真正的IO读写的时候需不需要用户线程参与。

使用NIO的代码如下

```java
// Selector的创建
Selector selector = Selector.open();
// Selector注册channel
channel.configureBlocking(false);
SelectionKey key = channel.register(selector,Selectionkey.OP_READ);
```
将channel与Selector配合使用。将channel注册到Selector上，使用registor()方法实现。使用Selector，Channel必须为非阻塞模式。 registor方法的第二个参数是Selector对那种Channel的时间感兴趣，可以监听不同的四种事件。Connect事件，Accept事件，Read事件，Write事件。
selectkey的合集

```java
SelectionKey.OP_CONNECT
SelectionKey.OP_ACCEPT
SelectionKey.OP_READ
SelectionKey.OP_WRITE
```

> 通过Selector来选择channel。一旦Selector注册了一个或者多个channel，就可以调用多个select方法。select()方法会阻塞到至少有一个通道在你注册的事件上就绪了。


```java
public class ServerHandler implements Runnable{
    private Selector selector;
    private ServerSocketChannel serverChannel;
    private volatile boolean started;

    public ServerHandler(int port) {
        try {
            selector = Selector.open();  //打开selector
            serverChannel = ServerSocketChannel.open();//打开channel
            serverChannel.configureBlocking(false); //channel为非阻塞
            serverChannel.bind(new InetSocketAddress(port), 1024); //绑定本地端口
            serverChannel.register(selector, SelectionKey.OP_ACCEPT); //将chaannel注册到selector中，这个channel专门监听连接请求
            started = true;
            System.out.println("服务器已启动，端口号：" + port);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public void stop(){
        started = false;
    }
    @Override
    public void run() {
        while (started) {

            try {
                selector.select(1000); //1s返回
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> it = keys.iterator();
                SelectionKey key = null;
                while(it.hasNext()){
                    key = it.next();
                    it.remove();
                    try{
                        handleInput(key);
                    }catch(Exception e){
                        if(key != null){
                            key.cancel();
                            if(key.channel() != null){
                                key.channel().close();
                            }
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (selector != null) {
            try {
                selector.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            //处理新接入的连接消息
            if (key.isAcceptable()) {
                ServerSocketChannel ssc = (ServerSocketChannel)key.channel();  //一个新的连接，则在服务端新建一个ServerSocketChannel专门来监听这个事件
                //通过ServerSocketChannel的accept创建SocketChannel实例
                //完成该操作意味着完成TCP三次握手，TCP物理链路正式建立
                SocketChannel sc = ssc.accept();
                //设置为非阻塞的
                sc.configureBlocking(false);
                //注册为读
                sc.register(selector, SelectionKey.OP_READ); 
            }
            if(key.isReadable()){
               //得到这个通道的信息 类似于指针    
               //连接已经建立。有一个socket抽象，那么建一个socketChannel来得到这个socket的引用
                SocketChannel sc = (SocketChannel) key.channel();  
                //创建ByteBuffer，并开辟一个1M的缓冲区
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                //读取请求码流，返回读取到的字节数
                int readBytes = sc.read(buffer); //主动去读取channel内的数据
                //读取到字节，对字节进行编解码
                if(readBytes>0){
                    //将缓冲区当前的limit设置为position=0，用于后续对缓冲区的读取操作
                    buffer.flip();
                    //根据缓冲区可读字节数创建字节数组
                    byte[] bytes = new byte[buffer.remaining()];
                    //将缓冲区可读字节数组复制到新建的数组中
                    buffer.get(bytes);
                    String expression = new String(bytes,"UTF-8");
                    System.out.println("服务器收到消息：" + expression);k
                    //处理数据
                    String result = null;
                    try{
                        result = expression.toString();
                    }catch(Exception e){
                        result = "计算错误：" + e.getMessage();
                    }
                    //发送应答消息
                    doWrite(sc,result);
                }
                //没有读取到字节 忽略
//              else if(readBytes==0);
                //链路已经关闭，释放资源
                else if(readBytes<0){
                    key.cancel();
                    sc.close();
                }
            }
        }
    }
    //异步发送应答消息
    private void doWrite(SocketChannel channel,String response) throws IOException{
        //将消息编码为字节数组
        byte[] bytes = response.getBytes();
        //根据数组容量创建ByteBuffer
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        //将字节数组复制到缓冲区
        writeBuffer.put(bytes);
        //flip操作
        writeBuffer.flip();
        //发送缓冲区的字节数组
        channel.write(writeBuffer);
        //****此处不含处理“写半包”的代码
    }
}
```

## 服务端建立一个NIO的过程

1. 打开ServerSocketChannel，监听客户端连接
2. 绑定监听端口，设置连接为非阻塞模式
3. 创建Reactor线程，创建多路复用器并启动线程
4. 将ServerSocketChannel注册到Reactor线程中的Selector上，监听ACCEPT事件
5. Selector轮询准备就绪的key
6. Selector监听到新的客户端接入，处理新的接入请求，完成TCP三次握手，简历物理链路
7. 设置客户端链路为非阻塞模式
8. 将新接入的客户端连接注册到Reactor线程的Selector上，监听读操作，读取客户端发送的网络消息
9. 异步读取客户端消息到缓冲区
10. 对Buffer编解码，处理半包消息，将解码成功的消息封装成Task
11. 将应答消息编码为Buffer，调用SocketChannel的write将消息异步发送给客户端


