---
title: java长连接实现原理初解
date: 2017-02-21 16:25:26
tags: [socket,长连接,tcp]
categories: Java
---

## 前言

一直觉得socket晦涩难懂，仅仅只是对着书敲了一下代码，实现了tcp的简单通信，但其中很多原理并不清楚。

## tcp简单实现

### 客户端（发送数据）

```java
//1、创建客户端Socket，指定服务器地址和端口
Socket socket =new Socket("127.0.0.1",9013);
//2、获取输出流，向服务器端发送信息
OutputStream os = socket.getOutputStream();//字节输出流
os.write("测试数据".getBytes());
os.flush();
socket.shutdownOutput();
os.close();
```

<!--more-->

### 服务端（接收数据）

```java
ServerSocket serverSocket=new ServerSocket(9013);
Socket socket = serverSocket.accept();
InputStream inputStream = socket.getInputStream();
byte[] buffer = new byte[1024];
int len = 0;
ByteArrayOutputStream bos = new ByteArrayOutputStream();
while ((len = inputStream.read(buffer)) != -1) {
bos.write(buffer, 0, len);
}
bos.close();
inputStream.close();
String s = new String(bos.toByteArray(), "UTF-8");
System.out.println(s);
```

这只是一次简单的通信，每一次发送数据后连接就断开了，想要再次发送就只能重新运行。那么，我想要实现长连接又该怎样实现呢？

## 改进

我在客户端通过一个`while`循环来模拟多次发送数据，服务端也同样使用循环来接收数据

```java
boolean b = true;
while(b){
  os.write("测试数据".getBytes());
  os.flush();
  socket.shutdownOutput();
  Thread.sleep(5000);
}
```

但结果是悲剧的，服务端只接收到一次数据，客户端发送第二次数据的时候报错了，原因是`socket`已经关闭。原来是`shutdownOutput`方法关闭了连接。

我注释掉这个方法后，服务端一直在等待状态，没有打印出东西来。

百度了很久我才明白了为什么加上`shutdownOutput`方法后，服务端能够打印数据。因为这个方法会给写入流加上结束符，而服务端就是判断读到了结束符就停止，没有的话就一直读，因为`read`方法是阻塞的，所以服务端一直在等待客户端给他发一个结束符过去。

明白了这个过后就好办了，我们把内容长度发过去，服务端就知道数据的长度了。

```java
String[] string = {"用户名：admin；密码：admin","身无彩凤双飞翼，心有灵犀一点通。","两情若是久长时，又岂在朝朝暮暮。"
            ,"沾衣欲湿杏花雨，吹面不寒杨柳风。","何须浅碧轻红色，自是花中第一流。","更无柳絮因风起，唯有葵花向日倾。"
            ,"海上生明月，天涯共此时。","一寸丹心图报国，两行清泪为思亲。","清香传得天心在，未话寻常草木知。",
            "和风和雨点苔纹，漠漠残香静里闻。"};
//1、创建客户端Socket，指定服务器地址和端口
Socket socket =new Socket("127.0.0.1",9013);
while (socket!=null){
  //2、获取输出流，向服务器端发送信息
  OutputStream os = socket.getOutputStream();//字节输出流
  for (String s:string) {
    byte[] bytes = s.getBytes();
    //
    os.write(int2ByteArrays(bytes.length));
    os.write(bytes);
    os.flush();
    Thread.sleep(5000);
  }
```

`int2ByteArrays`方法的作用是把数据的长度用四个字节来装

```java
public static byte[] int2ByteArrays(int i) {
        byte[] result = new byte[4];
        result[0] = (byte) ((i >> 24) & 0xFF);
        result[1] = (byte) ((i >> 16) & 0xFF);
        result[2] = (byte) ((i >> 8) & 0xFF);
        result[3] = (byte) (i & 0xFF);
        return result;
    }
```

服务端的读取数据部分也要改一下

```java
/**
 * 并没有关闭输入输出流
 *
 * @param inputStream
 * @return
 */
public static  String readFromStream(InputStream inputStream) {
  DataInputStream dis = null;

  try {
    dis = new DataInputStream(inputStream);
	//先得到数据长度
    byte[] header = new byte[4];
    int length = dis.read(header);
    int lengths = byteArrayToInt(header);
    //获取数据
    header = new byte[lengths];
    dis.read(header);
    String s = new String(header);
    System.out.println(length + "\t" + s);
    return s;
  } catch (IOException e) {
    e.printStackTrace();
  }
  return null;
}
```

这样就实现了一个简单的长连接。但，这还不够，服务端还应该实现在接收数据的同时也返回数据给客户端，客户端同理。那么，接收和发送需要做成两个独立的线程来实现。而且长连接还需要心跳机制来保活，所谓的心跳就是，每隔固定的时间，双方发一个固定的数据告诉对方：兄弟，我还在，你不要担心。



## 有问题反馈

在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

- Email: 2563892038@qq.com
- Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)