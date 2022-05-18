---
title: 手动搭建I/O网络通信框架2：BIO编程模型实现群聊
date: 2021-05-28 19:39:32
categories: Netty
tags: 
- netty
---

# 手动搭建I/O网络通信框架2：BIO编程模型实现群聊

> 转载自[https://www.cnblogs.com/lbhym/p/12681787.html](https://www.cnblogs.com/lbhym/p/12681787.html)

# **手动搭建I/O网络通信框架2：BIO编程模型实现群聊**

在第一章中运用Socket和ServerSocket简单的实现了网络通信。这一章，利用BIO编程模型进行升级改造，实现群聊聊天室。

所谓BIO，就是Block IO，阻塞式的IO。这个阻塞主要发生在：ServerSocket接收请求时（accept()方法）、InputStream、OutputStream（输入输出流的读和写）都是阻塞的。这个可以在下面代码的调试中发现，比如在客户端接收服务器消息的输入流处打上断点，除非服务器发来消息，不然断点是一直停在这个地方的。也就是说这个线程在这时间是被阻塞的。

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411190359629-713089288.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411190359629-713089288.png)

如图：当一个客户端请求进来时，接收器会为这个客户端分配一个工作线程，这个工作线程专职处理客户端的操作。在上一章中，服务器接收到客户端请求后就跑去专门服务这个客户端了，所以当其他请求进来时，是处理不到的。

看到这个图，很容易就会想到线程池，BIO是一个相对简单的模型，实现它的关键之处也在于线程池。

在上代码之前，先大概说清楚每个类的作用，以免弄混淆。更详细的说明，都写在注释当中。

## **服务器端：**

ChatServer:这个类的作用就像图中的Acceptor。它有两个比较关键的全局变量，一个就是存储在线用户信息的Map，一个就是线程池。这个类会监听端口，接收客户端的请求，然后为客户端分配工作线程。还会提供一些常用的工具方法给每个工作线程调用，比如：发送消息、添加在线用户等。我之前简单用过Netty和WebSocket，这个类看上去就已经和这些框架有点相似了。学习IO编程模型也是为了接下来深入学习Netty做准备。

ChatHandler:这个类就是工作线程的类。在这个项目中，它的工作很简单：把接收到的消息转发给其他客户端，当然还有一些小功能，比如添加\移除在线用户。

## **客户端：**

相较于服务器，客户端的改动较小，主要是把等待用户输入信息这个功能分到其他线程做，不然这个功能会一直阻塞主线程，导致无法接收其他客户端的消息。

ChatClient:客户端启动类，也就是主线程，会通过Socket和服务器连接。也提供了两个工具方法：发送消息和接收消息。

UserInputHandler:专门负责等待用户输入信息的线程，一旦有信息键入，就马上发送给服务器。

首先创建两个包区分一下客户端和服务器，client和server

**服务器端ChatServer：**

```java
public class ChatServer {
    private int DEFAULT_PORT = 8888;
    /**
     * 创建一个Map存储在线用户的信息。这个map可以统计在线用户、针对这些用户可以转发其他用户发送的消息
     * 因为会有多个线程操作这个map，所以为了安全起见用ConcurrentHashMap
     * 在这里key就是客户端的端口号，但在实际中肯定不会用端口号区分用户，如果是web的话一般用session。
     * value是IO的Writer，用以存储客户端发送的消息
     */
    private Map<Integer, Writer> map=new ConcurrentHashMap<>();
    /**
     * 创建线程池，线程上限为10个，如果第11个客户端请求进来，服务器会接收但是不会去分配线程处理它。
     * 前10个客户端的聊天记录，它看不见。当有一个客户端下线时，这第11个客户端就会被分配线程，服务器显示在线
     * 大家可以把10再设置小一点，测试看看
     * */
    private ExecutorService executorService= Executors.newFixedThreadPool(10);
    //客户端连接时往map添加客户端
    public void addClient(Socket socket) throws IOException {
        if (socket != null) {
            BufferedWriter writer = new BufferedWriter(
                    new OutputStreamWriter(socket.getOutputStream())
            );
            map.put(socket.getPort(), writer);
            System.out.println("Client["+socket.getPort()+"]:Online");
        }
    }

    //断开连接时map里移除客户端
    public void removeClient(Socket socket) throws Exception {
        if (socket != null) {
            if (map.containsKey(socket.getPort())) {
                map.get(socket.getPort()).close();
                map.remove(socket.getPort());
            }
            System.out.println("Client[" + socket.getPort() + "]Offline");
        }
    }

    //转发客户端消息，这个方法就是把消息发送给在线的其他的所有客户端
    public void sendMessage(Socket socket, String msg) throws IOException {
        //遍历在线客户端
        for (Integer port : map.keySet()) {
            //发送给在线的其他客户端
            if (port != socket.getPort()) {
                Writer writer = map.get(port);
                writer.write(msg);
                writer.flush();
            }
        }
    }

    //接收客户端请求，并分配Handler去处理请求
    public void start() {
        try (ServerSocket serverSocket = new ServerSocket(DEFAULT_PORT)) {
            System.out.println("Server Start,The Port is:"+DEFAULT_PORT);
            while (true){
                //等待客户端连接
                Socket socket=serverSocket.accept();
                //为客户端分配一个ChatHandler线程
                executorService.execute(new ChatHandler(this,socket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ChatServer server=new ChatServer();
        server.start();
    }
}
```

**服务器端ChatHandler：**

```java
public class ChatHandler implements Runnable {
    private ChatServer server;
    private Socket socket;

    //构造函数，ChatServer通过这个分配Handler线程
    public ChatHandler(ChatServer server, Socket socket) {
        this.server = server;
        this.socket = socket;
    }

    @Override
    public void run() {
        try {
            //往map里添加这个客户端
            server.addClient(socket);
            //读取这个客户端发送的消息
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
            );
            String msg = null;
            while ((msg = reader.readLine()) != null) {
                //这样拼接是为了让其他客户端也能看清是谁发送的消息
                String sendmsg = "Client[" + socket.getPort() + "]:" + msg;
                //服务器打印这个消息
                System.out.println(sendmsg);
                //将收到的消息转发给其他在线客户端
                server.sendMessage(socket, sendmsg + "\n");
                if (msg.equals("quit")) {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            //如果用户退出或者发生异常，就在map中移除该客户端
            try {
                server.removeClient(socket);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

**客户端ChatClient:**

```java
public class ChatClient {
    private BufferedReader reader;
    private BufferedWriter writer;
    private Socket socket;
    //发送消息给服务器
    public void sendToServer(String msg) throws IOException {
        //发送之前，判断socket的输出流是否关闭
        if (!socket.isOutputShutdown()) {
            //如果没有关闭就把用户键入的消息放到writer里面
            writer.write(msg + "\n");
            writer.flush();
        }
    }
    //从服务器接收消息
    public String receive() throws IOException {
        String msg = null;
        //判断socket的输入流是否关闭
        if (!socket.isInputShutdown()) {
            //没有关闭的话就可以通过reader读取服务器发送来的消息。注意：如果没有读取到消息线程会阻塞在这里
            msg = reader.readLine();
        }
        return msg;
    }

    public void start() {
        //和服务创建连接
        try {
            socket = new Socket("127.0.0.1", 8888);
            reader=new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
            );
            writer=new BufferedWriter(
                    new OutputStreamWriter(socket.getOutputStream())
            );
            //新建一个线程去监听用户输入的消息
            new Thread(new UserInputHandler(this)).start();
            /**
             * 不停的读取服务器转发的其他客户端的信息
             * 记录一下之前踩过的小坑：
             * 这里一定要创建一个msg接收信息，如果直接用receive()方法判断和输出receive()的话会造成有的消息不会显示
             * 因为receive()获取时，在返回之前是阻塞的，一旦接收到消息才会返回，也就是while这里是阻塞的，一旦有消息就会进入到while里面
             * 这时候如果输出的是receive(),那么上次获取的信息就会丢失，然后阻塞在System.out.println
             * */
            String msg=null;
            while ((msg=receive())!=null){
                System.out.println(msg);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
               if(writer!=null){
                   writer.close();
               }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        new ChatClient().start();
    }
}
```

　**客户端UserInputHandler:**

```java
public class UserInputHandler implements Runnable {
    private ChatClient client;

    public UserInputHandler(ChatClient client) {
        this.client = client;
    }

    @Override
    public void run() {
        try {
            //接收用户输入的消息
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(System.in)
            );
            //不停的获取reader中的System.in，实现了等待用户输入的效果
            while (true) {
                String input = reader.readLine();
                //向服务器发送消息
                client.sendToServer(input);
                if (input.equals("quit"))
                    break;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## **运行测试：**

通过打开终端，通过javac编译。如果大家是在IDEA上编码的话可能会报编码错误，在javac后面加上-encoding utf-8再接java文件就好了。

编译后运行，通过java运行时，又遇到了一个坑。会报找不到主类的错误，原来是因为加上两个包，要在class文件名前面加上包名。比如当前在src目录，下面有client和server两个包，要这么运行：java client.XXXX。可我之前明明在client文件夹下运行的java，也是不行，不知道为什么。

接着测试：

1.首先在一个终端里运行ChatServer，打开服务器

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411193234531-59946096.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411193234531-59946096.png)

2.在第二个终端里打开ChatClient，暂且叫A，此时服务器的终端显示：

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411193327017-1431424368.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411193327017-1431424368.png)

3.类似的，在第三个终端里打开ChatClient，暂且叫B，此时服务器显示：

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411193358577-1779741867.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411193358577-1779741867.png)

4.A中输入hi,除了服务器会打印hi外，B中也会显示，图片中的端口号和前面的不一样，是因为中间出了点小问题，前三张截图和后面的不是同时运行的。实际中同一个客户端会显示一样的端口号：

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411193747903-851197924.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411193747903-851197924.png)

5.当客户端输入quit时就会断开连接，最后，服务器的显示为：

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200411193923381-1365643546.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200411193923381-1365643546.png)