---
title: Netty初步了解
date: 2017-02-24 11:40:55
tags: [socket,netty,nio]
categories: Netty
---

## Netty是什么

Netty是由JBOSS提供的一个java开源框架。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。简单的来说，Netty 是一个基于NIO的客户、服务器端编程框架。

之前我们用java提供的原始i/o写了一个简易的长连接，程序很简单，逻辑也很清楚，但是却对服务器的压力很大，因为每一个socket都需要一个Thread来维持。

java 1.4以后引入了一个新的io —— new IO(简称nio)来代替原来的IO。nio的工作方式是基于通道(Channels)和缓冲区(Buffers)的，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中，并通过选择器(Selectors)来进行监听管理。这样就可以实现了一个线程去维持多个socket。

<!--more-->

我们之前说了，Netty是基于NIO的，他在原本比较复杂的NIO上简化和流线化了网络应用的编程开发过程，那么我们如果去使用它呢。

## 安装

先到[Netty官网](http://netty.io/downloads.html)上去下载，目前到我现在下载的时候，最新的Final版是：[netty-4.1.8.Final.tar.bz2](http://dl.bintray.com/netty/downloads/netty-4.1.8.Final.tar.bz2)，解压后直接把netty-4.1.8.Final\jar\all-in-one目录下的`netty-all-4.1.8.Final.jar`引入项目工程即可。

## 使用

### 客户端

```java
public class HelloClient {

    public static String host = "127.0.0.1";
    public static int port = 7878;

    /**
     * @param args
     * @throws InterruptedException
     * @throws IOException
     */
    public static void main(String[] args) throws InterruptedException, IOException {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new HelloInitializer());

            // 连接服务端
            Channel ch = b.connect(host, port).sync().channel();

            // 控制台输入
            BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
            for (;;) {
                String line = in.readLine();
                if (line == null) {
                    continue;
                }
                /*
                 * 向服务端发送在控制台输入的文本 并用"\r\n"结尾
                 * 之所以用\r\n结尾 是因为我们在handler中添加了 DelimiterBasedFrameDecoder 帧解码。
                 * 这个解码器是一个根据\n符号位分隔符的解码器。所以每条消息的最后必须加上\n否则无法识别和解码
                 * */
                ch.writeAndFlush(line + "\r\n");
            }
        } finally {
            // 优雅的关闭连接
            group.shutdownGracefully();
        }
    }
}
```

逻辑很简单，构造了`Bootstrap`对象后打开连接即可。前面我们说了数据是从`Buffer`到`Channel`,所以我们可以通过`channel()`方法得到一个`channel`，然后向`channel`中写入数据。

### 服务端

```java
public class HelloServer {
    /**
     * 服务端监听的端口地址
     */
    private static final int portNumber = 7878;

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup);
            b.channel(NioServerSocketChannel.class);
            b.childHandler(new HelloInitializer());

            // 服务器绑定端口监听
            ChannelFuture f = b.bind(portNumber).sync();
            // 监听服务器关闭监听
            f.channel().closeFuture().sync();

            // 可以简写为
            /* b.bind(portNumber).sync().channel().closeFuture().sync(); */
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

我们可以看到，服务端的逻辑实际上与客服端的大同小异的。只是它构造的不是普通的`Bootstrap`对象了，而是`ServerBootstrap`。

这还没有完，不管是客户端还是服务端，它们都使用了HelloInitializer这个类。这个类是一个初始化器，里面定义了数据的分割、解码、编码等等。

### HelloInitializer

```java
public class HelloInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        // 以("\n")为结尾分割的 解码器
        pipeline.addLast(new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));

        // 字符串解码 和 编码
      	// 这个地方的 必须和服务端对应上。否则无法正常解码和编码
        pipeline.addLast(new StringDecoder());
        pipeline.addLast(new StringEncoder());

        // 自己的逻辑Handler
        pipeline.addLast(new HelloHandler());
    }
}
```

### HelloHandler

```java
public class HelloHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println("对方说 : " + msg);
      	// 返回客户端消息 - 我已经接收到了你的消息
      	ctx.writeAndFlush("你说什么？大声点！\n");
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("设备被激活了");
        super.channelActive(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        ctx.close();
        System.out.println("设备退出了");
    }


}
```



## 有问题反馈

在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

- Email: 2563892038@qq.com
- Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)