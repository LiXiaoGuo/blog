---
title: socket实现长连接
date: 2017-02-22 16:22:36
tags: [socket,长连接,tcp]
categories: Java
---

上一篇文章我们简单的实现了一个长连接，但是还很不完善，连最基本的心跳机制都没有。接下来，我们继续完善我们的长连接。

### **基本数据协议**

通过上一篇文章我们知道我们得自己定义一个协议来让服务端知道什么时候数据读完了，那么我们先来完善这个基本协议。

<!--more-->

#### BasicProtocol：

```java
/**
 * 基本协议
 * Created by Extends on 2017/2/22.
 */
public abstract class BasicProtocol {
    static final int TYPE_LEN = 4;//表示业务类型；0000 -> 心跳包   0001 -> 发送普通文字消息
    static final int CONTEXT_LEN = 4;

    /**
     * 获取正文文本
     * @return
     */
    public abstract String getContext();

    /**
     * 获取包装好的byte[]
     * @return
     */
    public byte[] getData() {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            baos.write(getType().getBytes(),0,TYPE_LEN);
            byte[] bytes = getContext().getBytes();
            baos.write(ProtocolUtil.int2ByteArrays(bytes.length),0,CONTEXT_LEN);
            baos.write(bytes,0,bytes.length);
            return baos.toByteArray();
        }catch (Exception e){
            return null;
        }
    }

    /**
     * 获取业务类型
     * @return
     */
    public abstract  String getType();

    /**
     * 解析数据
     * @param bytes
     */
    public abstract void parseBinary(byte[] bytes);

}
```

这是一个抽象类，他的`byte[]`由三部分构成：第一部分是由四个字节表示的业务类型，比如说心跳包(0000),普通文字消息(0001)等等；第二部分是由四个字节表示的内容长度；第三部分就是内容了。我这里定义好基本协议后，以后的所有协议都要满足这个条件，比如下面的两个类。

#### HeartBeatProtocol：

```java
/**
 * 心跳包
 * Created by Extends on 2017/2/22.
 */
public class HeartBeatProtocol extends BasicProtocol {
    static final String TYPE = "0000";
    @Override
    public String getContext() {
        return "兄弟，我还在，你不要担心";
    }

    @Override
    public String getType() {
        return TYPE;
    }

    @Override
    public void parseBinary(byte[] bytes) {

    }
}
```

#### MessageProtocol：

```java
/**
 * 普通文字消息
 * Created by Extends on 2017/2/22.
 */
public class MessageProtocol extends BasicProtocol {

    private String context;
    static final String TYPE = "0001";

    public void setContext(String context){
        this.context = context;
    }

    @Override
    public String getContext() {
        return context;
    }

    @Override
    public String getType() {
        return TYPE;
    }

    @Override
    public void parseBinary(byte[] bytes) {
        setContext(new String(bytes));
    }
}
```

数据协议暂时就这些，接着再来一个简单的工具类来帮助我们更方便的使用它们。

#### ProtocolUtil：

```java
/**
 * 协议工具类
 * Created by Extends on 2017/2/22.
 */
public class ProtocolUtil {

    private static Map<String,String> msgImp=new HashMap<String,String>();
    static {
        msgImp.put(HeartBeatProtocol.TYPE,"protocol.HeartBeatProtocol");
        msgImp.put(MessageProtocol.TYPE,"protocol.MessageProtocol");
    }

    /**
     * 返回业务类型
     * @param data
     * @return
     */
    public static String paraseType(byte[] data){
        return new String(data,0,BasicProtocol.TYPE_LEN);
    }

    /**
     * int 转 四位字节
     * @param i
     * @return
     */
    public static byte[] int2ByteArrays(int i) {
        byte[] result = new byte[4];
        result[0] = (byte) ((i >> 24) & 0xFF);
        result[1] = (byte) ((i >> 16) & 0xFF);
        result[2] = (byte) ((i >> 8) & 0xFF);
        result[3] = (byte) (i & 0xFF);
        return result;
    }

    /**
     * 字节数组转int
     * @param b
     * @return
     */
    public static int byteArrayToInt(byte[] b) {
        int intValue = 0;
        for (int i = 0; i < b.length; i++) {
            intValue += (b[i] & 0xFF) << (8 * (3 - i));
        }
        return intValue;
    }

    /**
     * 从输入流中读取数据
     * @param inputStream
     * @return
     */
    public static BasicProtocol readInputStream(DataInputStream inputStream) {
        try {
            byte[] bytes = new byte[BasicProtocol.TYPE_LEN];
            int type_len = inputStream.read(bytes);
            if(type_len!=bytes.length){
                return null;
            }
            String s_type = paraseType(bytes);
            bytes = new byte[BasicProtocol.CONTEXT_LEN];
            int context_len = inputStream.read(bytes);
            if(context_len != bytes.length){
                return null;
            }
            bytes = new byte[byteArrayToInt(bytes)];
            inputStream.read(bytes);

            BasicProtocol basicProtocol = (BasicProtocol) Class.forName(msgImp.get(s_type)).newInstance();
            basicProtocol.parseBinary(bytes);
            return basicProtocol;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 向输出流中写入数据
     * @param protocol
     * @param outputStream
     */
    public static boolean writeOutputStream(BasicProtocol protocol,DataOutputStream outputStream) {
        try {
            outputStream.write(protocol.getData());
            outputStream.flush();
            return true;
        } catch (IOException e) {
            e.printStackTrace();
            return false;
        }
    }
}
```

到此为止，基本数据协议我们已经写好了，可以说，最重要的部分我们已经完成了，剩下的就只是搭搭积木。

## **基本架构：**

不管是服务端还是客户端，他们都要收发数据，且两个事件相对独立，那么我们需要两个不同的线程来分别执行收发逻辑。读数据没什么难度，但是写数据的话还得考虑到每隔固定的时间发一个心跳包，并且不时的还有普通数据发过去，那么就需要一个线程安全的消息队列来做数据的发放了。这么一想，其实就很简单了。

#### LongServer：

```java
public class LongServer implements Runnable {
    private static String[] string = {"用户名：admin；密码：admin","身无彩凤双飞翼，心有灵犀一点通。","两情若是久长时，又岂在朝朝暮暮。"
            ,"沾衣欲湿杏花雨，吹面不寒杨柳风。","何须浅碧轻红色，自是花中第一流。","更无柳絮因风起，唯有葵花向日倾。"
            ,"海上生明月，天涯共此时。","一寸丹心图报国，两行清泪为思亲。","清香传得天心在，未话寻常草木知。",
            "和风和雨点苔纹，漠漠残香静里闻。"};

    //消息队列
    private volatile ConcurrentLinkedQueue<BasicProtocol> reciverData= new ConcurrentLinkedQueue<BasicProtocol>();
    private WriteTask writeTask;//写数据的线程
    private ReadTask readTask;//读数据的线程
    private Socket socket;

    public LongServer(Socket socket){
        this.socket = socket;
    }

    @Override
    public void run() {
        try {
            readTask = new ReadTask();
            writeTask = new WriteTask();


            writeTask.outputStream = new DataOutputStream(socket.getOutputStream());//默认初始化发给自己
            readTask.inputStream = new DataInputStream(socket.getInputStream());


            readTask.start();
            writeTask.start();


        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 负责读取数据
     */
    public class ReadTask extends Thread{
        private DataInputStream inputStream;
        private boolean isCancle = false;//是否取消循环
        @Override
        public void run() {
            try {
                while (!isCancle){
                    BasicProtocol protocol = ProtocolUtil.readInputStream(inputStream);
                    if(protocol != null){
                        System.out.println("================:"+protocol.getContext());
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
                stops();//捕获到io异常，可能原因是连接断开了，所以我们停掉所有操作
            }
        }
    }

    /**
     * 负责写入数据
     */
    public class WriteTask extends Thread{
        private DataOutputStream outputStream;
        private boolean isCancle = false;
        private Timer heart = new Timer();//发送心跳包的定时任务
        private Timer message = new Timer();//模拟发送普通数据
        @Override
        public void run() {
            //每隔20s发送一次心跳包
            heart.schedule(new TimerTask() {
                @Override
                public void run() {
                    reciverData.add(new HeartBeatProtocol());
                }
            },0,1000*20);

            //先延时2s，然后每隔6s发送一次普通数据
            Random random = new Random();
            message.schedule(new TimerTask() {
                @Override
                public void run() {
                    MessageProtocol bp = new MessageProtocol();
                    bp.setContext(string[random.nextInt(string.length)]);
                    reciverData.add(bp);
                }
            },1000*2,1000*6);


            while (!isCancle){
                BasicProtocol bp = reciverData.poll();
                if(bp!=null){
                    System.out.println("------:"+bp.getContext());
                    ProtocolUtil.writeOutputStream(bp,outputStream);
                }
            }
        }
    }

    /**
     * 停止掉所有活动
     */
    public void stops(){
        if (readTask!=null){
            readTask.isCancle=true;
            readTask.interrupt();
            readTask=null;
        }

        if (writeTask!=null) {
            writeTask.isCancle = true;
            //取消发送心跳包的定时任务
            writeTask.heart.cancel();
            //取消发送普通消息的定时任务
            writeTask.message.cancel();
            writeTask.interrupt();
            writeTask=null;
        }
    }
}
```

#### main：

核心代码已经完全写出来了，只缺客户端和服务端的main方法了。客户端好说，只需要把socket传进去就可以了；服务端的话可以考虑一下多个用户的长连接。我们可以这样：

```java
ublic static void main(String[] args){

    ServerSocket serverSocket=null;
    ExecutorService executorService=Executors.newCachedThreadPool();
    try {
        serverSocket=new ServerSocket(9013);
        while (isStart) {
            Socket socket = serverSocket.accept();
            String userIP=socket.getInetAddress().getHostAddress();
            System.out.println("用户的IP地址为：" + userIP);
            serverResponseTask = new LongServer(socket);

            if (socket.isConnected()) {
                executorService.execute(serverResponseTask);
            }
        }
        serverSocket.close();
    } catch (IOException e) {
        e.printStackTrace();
    }finally {
        if (serverSocket!=null){
            try {
                isStart=false;
                serverSocket.close();
                if(serverSocket!=null)
                    serverResponseTask.stops();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```



## 有问题反馈

在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

- Email: 2563892038@qq.com
- Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)