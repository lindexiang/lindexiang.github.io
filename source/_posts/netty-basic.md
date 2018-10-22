---
title: netty基础
date: 2018-10-21 12:40:15
tags: 
    - netty基础
    - 粘包和拆包
    - NIO模型
    - socket编程
categories: socket编程
image: http://pbhb4py13.bkt.clouddn.com/2018-08-30-david-beatz-798135-unsplash.jpg
top : 1

---

# netty基础

## handler(处理器)
netty中的SimpleChannelInboundHandler继承了ChannelInboundHandler接口，这个接口提供了很多处理事件的接口方法，可以覆盖这些方法来实现自己的逻辑。
<!-- more -->

* handlerAdded 服务端收到新的客户端连接，会调用该方法
* handlerRevmoved 服务端收到客户端断开，调用该方法
* channelRead0  服务端读到客户端的写入信息
* channelActive 服务端监听客户端的活动
* channelInactive  客户端不在线
* exceptiponCause 当出现Throwable对象才会调用，当Netty出现IO错误或者处理器抛出异常。记录错误信息将有问题的channel关闭。

```java
public class SimpleChatServerHandler extends SimpleChannelInboundHandler<String>{

    public static ChannelGroup channels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE); //所有的通道

    /**
    * 有新的客户端接入，调用该方法
    * */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Channel incoming = ctx.channel(); // 每次有handler加入，就会生成ctx的channel
        for (Channel channel : channels) {
            channel.writeAndFlush("[SERVER] - " + incoming.remoteAddress() + " 加入\n");
        }
        channels.add(incoming);
    }
    /**
    * 有客户端离开，调用该方法
    * */
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Channel incoming = ctx.channel(); // 每次有handler加入，就会生成ctx的channel
        for (Channel channel : channels) {
            channel.writeAndFlush("[SERVER] - " + incoming.remoteAddress() + " 离开\n");
        }
        channels.remove(incoming);
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String s) throws Exception {
        Channel incoming = ctx.channel();
        for (Channel channel : channels) {
            if (channel != incoming) {
                channel.writeAndFlush("[" + incoming.remoteAddress() + "]" + s + "\n");
            } else {
                channel.writeAndFlush("[you]" + s + "\n");
            }
        }
    }
    /**
    * 在线检测
    * */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:" + incoming.remoteAddress() + "在线");
    }

    /**
    * 掉线检测
    * */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception { // (6)
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:"+incoming.remoteAddress()+"掉线");
    }

    /**
    * 异常检测
    * */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (7)
        Channel incoming = ctx.channel();
        System.out.println("SimpleChatClient:"+incoming.remoteAddress()+"异常");
        // 当出现异常就关闭连接
        cause.printStackTrace();
        ctx.close();
    }
```

## netty的粘包和拆包问题
服务端收到2个数据包，两条数据包都是完整的。
![30DCEEFA-1D9E-4DF0-A432-7958FCF77281](http://pbhb4py13.bkt.clouddn.com/2018-10-22-30DCEEFA-1D9E-4DF0-A432-7958FCF77281.jpg)

1. 粘包 
服务端一共就收到一个数据包，数据包中有2条消息，发生了TCP粘包
2. 拆包
一条消息被拆分到2个包里发送了

发生TCP的粘包和拆包的主要原因是 

1. 应用程序写入的数据大于套接字缓冲区的大小，发生了拆包
2. 应用程序写入的数据小于缓冲区的大小，网卡将应用多次写入的数据发送到网络中，发生粘包
3. 进行MSS大小的TCP分段，当TCP报文长度大于MSS，发生拆包
4. 接收方法不及时读取缓冲区大小，发生粘包

 如何解决粘包、拆包问题：
 
1. 使用带有消息头的协议，从消息头中读取出消息长度信息和开始标识。
2. 设置定长信息，服务端每次读取既定长度的内容
3. 设置消息边界，设置回车等等。


