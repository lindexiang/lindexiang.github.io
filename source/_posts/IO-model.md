---
title: LINUX的IO模型
date: 2018-10-21 19:40:15
tags: 
    - IO模型
    - BIO
    - NIO
    - AIO
    - 同步
    - 异步
    - 阻塞
    - 非阻塞
    - socket编程
categories: socket编程
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/minimalistic-landscape-mountains-forest-bird-sky-artwork-others-13757-1.jpg
top : 100

---
# LINUX的5种IO模型
## 简介
同步阻塞，同步非阻塞，异步阻塞，异步非阻塞这些不同的IO模型是如何定义的。在看了几篇博客后我总结了下，把自己说服通了。
服务端的应用程序要响应客户端的请求时，要有IO请求。在网络IO中，首先，内核会先收集完整的TCP或者UDP数据包，再将数据从内核复制到用户的内存中。这里的IO模型是针对于服务器端的，因为客户端一般都是阻塞的。
<!-- more -->
**操作系统是分为用户空间和内核空间。内核空间控制具体的硬件设备来获取网络中的IO请求和请求数据。用户的程序是通过系统调用来获取内核中的IO请求数据。**
**对于一个网络IO的情况，它会涉及到两个系统对象，一个是调用这个IO的进程或者线程，另一个就是系统内核(kernel)**。当一个read操作发生时，它会经历两个阶段：

1. 等待数据准备(waiting for the data to be ready)
2. 将数据从内核拷贝到进程中(copy the date from kernel to the process)

记住这两点很重要，因为这些IO Model的区别就是在两个阶段上各有不同的情况。
### IO模型
UNIX提供了5种的IO模型，任何的IO请求一定是这5种中的一种。
## 阻塞IO模型 (blocking IO)
在默认的情况下，BIO的操作均是阻塞的。
比如I/O模型下的套接字接口，进程空间recvfrom类似于`socket.accept()`，其系统调用直到数据包到达并且被复制到应用进程的缓冲区或者错误时才返回，在此期间一直等待。
进程在调用recvform开始直到返回的整段时间都是被阻塞的。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522306820154.jpg)

当用户进程recvfrom这个系统调用时候，kernal就开始IO的第一个阶段：准备数据。对于network io来说，很多时候数据还没有到达（比如没有收到完整的UDP包），kerna要等到收到足够的数据。而在用户进程这边，整个进程都会被阻塞。当kernal把数据准备好，它会将数据从kernal拷贝到用户内存中，然后kernal返回结果，用户进程菜解除block状态。
BIO在执行IO的两个阶段(等待数据和拷贝数据)都被block了。
**BIO阻塞IO有一个缺点就是用户线程在系统调用没有返回结果时会一直阻塞，只有系统调用超时或者错误才会返回。在此期间，线程将无法执行任何运算或者响应任何网络请求。**
一般BIO采用多线程让每个连接拥有独立的线程，这样任何连接的阻塞不会影响到其他的连接。当有上万个请求连接时，无论多线程才是多进程都会严重占用系统资源，降低响应效率。线程容易进入假死状态。**可以考虑采用多线程的方式，调整线程池的大小。**
**采用多线程模型可以方便高效的解决小规模的服务请求，但是大规模的服务请求时，多线程模型会遇到瓶颈，可以尝试采用非阻塞接口来解决问题。**
     
## 非阻塞IO模型 (non-blocking IO)
用户进程发出recv()操作时，如果kernel中的数据没有准备好，不会阻塞用户进程，立即返回一个error。从用户进程的角度看，它发起一个recvfrom操作后立即得到一个结果，当用户进程得到一个error后知道数据没有准备好，可以再次发起read操作。一旦kernel中的数据准备好，并且收到system call，马上将数据从内核拷贝到用户空间，然后返回。
**所以，在非阻塞IO中，用户进程需要不断的主动询问kernel数据是否准备好。**
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522324463630.jpg)
非阻塞接口和阻塞接口的区别是系统调用后立即返回，在非阻塞IO模型中，recvfrom接口在调用后立即返回，返回值代表不同的含义

* recv()返回值大于0，表示接收数据完毕，返回值是接收到的字节数
* recv()返回0，表示连接断开
* recv()返回-1，且errno等于EAGAIN，表示recv操作还没执行完成
* recv()返回-1，且errno不等于EAGAIN，表示recv遇到系统错误。

服务器线程通过循环调用recv()接口，可以在单个线程内实现对所有连接的接收工作，但是循环调用recv()将大幅度推高CPU的占用率，**在这个方案中recv()的作用是检测“操作是否完成”的作用，操作系统有更为高效的方式检测“操作是否完成”**，比如select()多路复用模式，可以一次检测多个连接是否有效。

## 多路复用IO模型(IO multiplexing)
在非阻塞IO模型中，需要用户线程一直轮询recv()查看是否有数据准备好的连接，线程一直是活跃的，会浪费大量的CPU时间。
可以采用IO复用模型，即select/epoll方式，或者称为事件驱动IO模型。在select/epoll中，一个进程可以处理多个网络连接IO，原理是select/epoll会不断轮询所负责的所有socket，当某个socket有数据到达时，通知用户进程，流程如下所示
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522340282798.jpg)
当用户进程(或线程)调用select,那么整个进程就被block了，同时kernel会“监视”所有select负责的socket，任何一个socket数据准备好了，select就会返回，这时用户进程再调用read操作，将数据从kernel拷贝到用户空间。
这个图和BIO模型没有太多的区别，并且需要使用2个系统调用(select和recvfrom)，而BIO只需要一个系统调用。但是，用select的优势是可以处理多个connection。
**在连接的数量不是很多的情况下，采用select/epoll的web server不一定比使用多线程模式的BIO模型的效率更高，可能延迟更加的严重，select/epoll的优势不是对单个连接处理更快，而是能处理更多的连接。**
在多路复用模型中，每一个socket都会被设置成non-blocking，整个用户进程process都是被阻塞的，只不过process被阻塞在select这个函数，而不是socket io阻塞，因此select()和BIO相似。
select()发现某个句柄捕捉到了“可读事件”，服务端会及时作出recv()操作，当句柄发现了“可写事件”，程序及时作出send()操作。下面就是select模型的一个执行周期。
 ![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522360061310.jpg)
相比其他模型，采用select()的事件驱动模型采用单线程，占用资源少，不消耗很多的cpu，同时为多个客户端提供服务。但是该模型存在很多问题。
当探测的句柄值较大时，select()接口本身就会消耗大量的时间去轮询各个句柄，很多操作系统提供更高级的接口，比如linux提供了epoll。
## 异步IO(asynchronous IO)
linux下的asynchronous IO其实用的不多，在linux的2.6版本才引入。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522366996884.jpg)
用户进程发起read操作后，立即可以开始做其他的事情。从kernel的角度看，受到一个asynchronous read后，首先立即返回，不会对用户进程产生任何block。然后kernel会等待数据准备完成，然后将数据从内核拷贝到用户内存。最后，kernel给用户进程发送一个signal，告诉它read操作完成了。异步IO是真正的非阻塞的。

## IO阻塞，非阻塞，同步和异步概念

1. 阻塞和非阻塞  
调用阻塞IO会一直block住对应的进程直到操作完成，非阻塞IO在kernel还在准备数据的情况下会立即返回。
2. 同步和异步
POSIX中对synchronous IO和asynchronous IO的定义如下

* a synchronous IO operation causes the requesting process to be blocked until IO operation completes;
* an asynchronous IO operation does not cause the requesting process to be blocked;
其中，operation IO指的是真实的IO操作，所以阻塞IO，非阻塞IO，IO复用都是属于同步IO，就是recvfrom这个系统调用。
同步IO在kernel数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存，这时候进程是被block了，而异步IO在进程发起IO操作后，直接返回直到kernel发出signal告诉进程IO完成。

各个IO的模型如下所示
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15522717138750.jpg)





