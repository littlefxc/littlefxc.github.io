---
title: 手动搭建I/O网络通信框架4：AIO编程模型，聊天室终极改造
date: 2021-05-28 19:39:32
categories: Netty
tags: 
- netty
---

# 手动搭建I/O网络通信框架4：AIO编程模型，聊天室终极改造

> 转载自 [https://www.cnblogs.com/lbhym/p/12720944.html](https://www.cnblogs.com/lbhym/p/12720944.html)

# 前言

上一章讲到的NIO编程模型比较主流，非常著名的Netty就是基于NIO编程模型的。这一章说的是AIO编程模型，是**异步非阻塞**的。虽然同样实现的是聊天室功能，但是实现逻辑上稍微要比NIO和BIO复杂一点。不过理好整体脉络，会好理解一些。首先还是讲讲概念：

BIO和NIO的区别是阻塞和非阻塞，而AIO代表的是异步IO。在此之前只提到了阻塞和非阻塞，没有提到异步还是同步。可以用我在知乎上看到的一句话表示：【在处理 IO 的时候，阻塞和非阻塞都是同步 IO，只有使用了特殊的 API 才是异步 IO】。这些“特殊的API”下面会讲到。在说AIO之前，先总结一下阻塞非阻塞、异步同步的概念。

**阻塞和非阻塞，描述的是结果的请求**。**阻塞**：在得到结果之前就一直呆在那，啥也不干，此时线程挂起，就如其名，线程被阻塞了。**非阻塞**：如果没得到结果就返回，等一会再去请求，直到得到结果为止。**异步和同步，描述的是结果的发出**，当调用方的请求进来。**同步**：在没获取到结果前就不返回给调用方，如果调用方是阻塞的，那么调用方就会一直等着。如果调用方是非阻塞的，调用方就会先回去，等一会再来问问得到结果没。**异步**：调用方一来，会直接返回，等执行完实际的逻辑后在通过回调函数把结果返回给调用方。

# **AIO中的异步操作**

## **CompletionHandler**

在AIO编程模型中，常用的API，如connect、accept、read、write都是支持异步操作的。当调用这些方法时，可以携带一个**CompletionHandler**参数，它会提供一些回调函数。这些回调函数包括:

1. 当这些操作成功时你需要怎么做；
2. 如果这些操作失败了你要这么做。

关于这个CompletionHandler参数，你只需要写一个类实现CompletionHandler口，并实现里面两个方法就行了。

那如何在调用connect、accept、read、write这四个方法时，传入CompletionHandler参数从而实现异步呢？下面分别举例这四个方法的使用。

先说说`Socket`和`ServerSocket`，在NIO中，它们变成了通道，配合缓冲区，从而实现了非阻塞。而在AIO中它们变成了异步通道。也就是`AsynchronousServerSocketChannel`和`AsynchronousSocketChannel`,下面例子中对象名分别是serverSocket和socket.

- accept：serverSocket.accept(attachment,handler)。

    handler就是实现了CompletionHandler接口并实现两个回调函数的类，它具体怎么写可以看下面的实战代码。attachment为handler里面可能需要用到的辅助数据，如果没有就填null。

- read：socket.read(buffer,attachment,handler)。

    buffer是缓冲区，用以存放读取到的信息。后面两个参数和accept一样。

- write：socket.write(buffer,attachment,handler)。

    和read参数一样。

- connect：socket.connect(address,attachment,handler)。

    address为服务器的IP和端口，后面两个参数与前几个一样。

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200417161130725-1186191947.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200417161130725-1186191947.png)

## **Future**

既然说到了异步操作，除了使用实现CompletionHandler接口的方式，不得不想到**Future**。客户端逻辑较为简单，如果使用CompletionHandler的话代码反而更复杂，所以下面的实战客户端代码就会使用Future的方式。简单来说，Future表示的是异步操作未来的结果，怎么理解未来。比如，客户端调用read方法获取服务器发来得消息：

```java
Future<Integer> readResult=clientChannel.read(buffer)
```

Integer是read()的返回类型，此时变量readResult实际上并不一定有数据，而是表示read()方法未来的结果，这时候readResult有两个方法，**isDone**()：返回boolean，查看程序是否完成处理，如果返回true，有结果了，这时候可以通过get()获取结果。如果你不事先判断isDone()直接调用**get**()也行，只不过它是阻塞的。如果你不想阻塞，想在这期间做点什么，就用isDone()。

还有一个问题：**这些handler的方法是在哪个线程执行的？**serverSocket.accept这个方法肯定是在主线程里面调用的，而传入的这些回调方法其实是在其他线程执行的。在AIO中，会有一个**AsynchronousChannelGroup**，它和AsynchronousServerSocketChannel是绑定在一起的，**它会为这些异步通道提供系统资源，线程就算其中一种系统资源**，所以为了方便理解，我们暂时可以把他看作一个线程池，它会为这些handler分配线程，而不是在主线程中去执行。

# **AIO编程模型**

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200417161442138-580814692.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200417161442138-580814692.png)

上面只说了些零碎的概念，为了更好的理解，下面讲一讲大概的工作流程(主要针对服务器，客户端逻辑较为简单，代码注释也比较少，可以看前面几章)：

1. 首先做准备工作。跟NIO一样，先要创建好通道，只不过AIO是异步通道。然后创建好AsyncChannelGroup，可以选择自定义线程池。最后把AsyncServerSocket和AsyncChannelGroup绑定在一起，这样处于同一个AsyncChannelGroup里的通道就可以共享系统资源。
2. 最后一步准备工作，创建好handler类，并实现接口和里面两个回调方法。（如图：客户端1对应的handler,里面的回调方法会实现读取消息和转发消息的功能；serverSocket的handler里的回调方法会实现accept功能。）
3. 准备工作完成，当客户端1连接请求进来，客户端会马上回去，ServerSocket的异步方法会在连接成功后把客户端的SocketChannel存进在线用户列表，并利用客户端1的handler开始异步监听客户端1发送的消息。
4. 当客户端1发送消息时，如果上一步中的handler成功监听到，就会回调成功后的回调方法，这个方法里会把这个消息转发给其他客户端。转发完成后，接着利用handler监听客户端1发送的消息。

代码一共有三个类：

- ChatServer：功能基本上和上面讲的工作流程差不多，还会有一些工具方法，都比较简单，就不多说了，如：转发消息，客户端下线后从在线列表移除客户端等。
- ChatClient：基本和前两章的BIO、NIO没什么区别，一个线程监听用户输入信息并发送，主线程异步的读取服务器信息。
- UserInputHandler：监听用户输入信息的线程。

## ChatServer

```java
public class ChatServer {
    //设置缓冲区字节大小
    private static final int BUFFER = 1024;

    //声明AsynchronousServerSocketChannel和AsynchronousChannelGroup
    private AsynchronousServerSocketChannel serverSocketChannel;
    private AsynchronousChannelGroup channelGroup;

    //在线用户列表。为了并发下的线程安全，所以使用CopyOnWriteArrayList
    //CopyOnWriteArrayList在写时加锁，读时不加锁，而本项目正好在转发消息时需要频繁读取.
    //ClientHandler包含每个客户端的通道，类型选择为ClientHandler是为了在write的时候调用每个客户端的handler
    private CopyOnWriteArrayList<ClientHandler> clientHandlerList;
    //字符和字符串互转需要用到，规定编码方式，避免中文乱码
    private Charset charset = Charset.forName("UTF-8");

    //通过构造函数设置监听端口
    private int port;
    public ChatServer(int port) {
        this.port = port;
        clientHandlerList=new CopyOnWriteArrayList<>();
    }

    public void start() {
        try {
            /**
             *创建一个线程池并把线程池和AsynchronousChannelGroup绑定，前面提到了AsynchronousChannelGroup包括一些系统资源，而线程就是其中一种。
             *为了方便理解我们就暂且把它当作线程池，实际上并不止包含线程池。如果你需要自己选定线程池类型和数量，就可以如下操作
             *如果不需要自定义线程池类型和数量，可以不用写下面两行代码。
             * */
            ExecutorService executorService = Executors.newFixedThreadPool(10);
            channelGroup = AsynchronousChannelGroup.withThreadPool(executorService);
            serverSocketChannel=AsynchronousServerSocketChannel.open(channelGroup);
            serverSocketChannel.bind(new InetSocketAddress("127.0.0.1",port));
            System.out.println("服务器启动：端口【"+port+"】");
            /**
             * AIO中accept可以异步调用，就用上面说到的CompletionHandler方式
             * 第一个参数是辅助参数，回调函数中可能会用上的，如果没有就填null;第二个参数为CompletionHandler接口的实现
             * 这里使用while和System.in.read()的原因：
             * while是为了让服务器保持运行状态，前面的NIO，BIO都有用到while无限循环来保持服务器运行，但是它们用的地方可能更好理解
             * System.in.read()是阻塞式的调用，只是单纯的避免无限循环而让accept频繁被调用，无实际业务功能。
             */
            while (true) {
                serverSocketChannel.accept(null, new AcceptHandler());
                System.in.read();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            if(serverSocketChannel!=null){
                try {
                    serverSocketChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    //AsynchronousSocketChannel为accept返回的类型，Object为辅助参数类型，没有就填Object
    private class AcceptHandler implements CompletionHandler<AsynchronousSocketChannel,Object>{
        //如果成功，执行的回调方法
        @Override
        public void completed(AsynchronousSocketChannel clientChannel, Object attachment) {
            //如果服务器没关闭，在接收完当前客户端的请求后，再次调用,以接着接收其他客户端的请求
            if(serverSocketChannel.isOpen()){
                serverSocketChannel.accept(null,this);
            }
            //如果客户端的channel没有关闭
            if(clientChannel!=null&&clientChannel.isOpen()){
                //这个就是异步read和write要用到的handler,并传入当前客户端的channel
                ClientHandler handler=new ClientHandler(clientChannel);
                //把新用户添加到在线用户列表里
                clientHandlerList.add(handler);
                System.out.println(getPort(clientChannel)+"上线啦！");
                ByteBuffer buffer=ByteBuffer.allocate(BUFFER);
                //异步调用read,第一个buffer是存放读到数据的容器，第二个是辅助参数。
                //因为真正的处理是在handler里的回调函数进行的，辅助参数会直接传进回调函数，所以为了方便使用，buffer就当作辅助参数
                clientChannel.read(buffer,buffer,handler);
            }
        }
        //如果失败，执行的回调方法
        @Override
        public void failed(Throwable exc, Object attachment) {
            System.out.println("连接失败"+exc);
        }
    }

    private class ClientHandler implements CompletionHandler<Integer, ByteBuffer>{
        private AsynchronousSocketChannel clientChannel;
        public ClientHandler(AsynchronousSocketChannel clientChannel) {
            this.clientChannel = clientChannel;
        }
        @Override
        public void completed(Integer result, ByteBuffer buffer) {
            if(buffer!=null){
                //如果read返回的结果小于等于0，而buffer不为空，说明客户端通道出现异常，做下线操作
                if(result<=0){
                    removeClient(this);
                }else {
                    //转换buffer读写模式并获取消息
                    buffer.flip();
                    String msg=String.valueOf(charset.decode(buffer));
                    //在服务器上打印客户端发来的消息
                    System.out.println(getPort(clientChannel)+msg);
                    //把消息转发给其他客户端
                    sendMessage(clientChannel,getPort(clientChannel)+msg);
                    buffer=ByteBuffer.allocate(BUFFER);

                    //如果用户输入的是退出，就从在线列表里移除。否则接着监听这个用户发送消息
                    if(msg.equals("quit"))
                        removeClient(this);
                    else
                        clientChannel.read(buffer, buffer, this);
                }
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            System.out.println("客户端读写异常："+exc);
        }
    }

    //转发消息的方法
    private void sendMessage(AsynchronousSocketChannel clientChannel,String msg){
        for(ClientHandler handler:clientHandlerList){
            if(!handler.clientChannel.equals(clientChannel)){
                ByteBuffer buffer=charset.encode(msg);
                //write不需要buffer当辅助参数，因为写到客户端的通道就完事了，而读还需要回调函数转发给其他客户端。
                handler.clientChannel.write(buffer,null,handler);
            }
        }
    }
    //根据客户端channel获取对应端口号的方法
    private String getPort(AsynchronousSocketChannel clientChannel){
        try {
            InetSocketAddress address=(InetSocketAddress)clientChannel.getRemoteAddress();
            return "客户端["+address.getPort()+"]:";
        } catch (IOException e) {
            e.printStackTrace();
            return "客户端[Undefined]:";
        }
    }
    //移除客户端
    private void removeClient(ClientHandler handler){
        clientHandlerList.remove(handler);
        System.out.println(getPort(handler.clientChannel)+"断开连接...");
        if(handler.clientChannel!=null){
            try {
                handler.clientChannel.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        new ChatServer(8888).start();
    }
}
```

## ChatClient

```java
public class ChatClient {
    private static final int BUFFER = 1024;
    private AsynchronousSocketChannel clientChannel;
    private Charset charset = Charset.forName("UTF-8");

    private String host;
    private int port;
    //设置服务器IP和端口
    public ChatClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void start() {
        try {
            clientChannel = AsynchronousSocketChannel.open();
            //连接服务器
            Future<Void> future = clientChannel.connect(new InetSocketAddress(host, port));
            future.get();
            //新建一个线程去等待用户输入
            new Thread(new UserInputHandler(this)).start();
            ByteBuffer buffer=ByteBuffer.allocate(BUFFER);
            //无限循环让客户端保持运行状态
            while (true){
                //获取服务器发来的消息并存入到buffer
                Future<Integer> read=clientChannel.read(buffer);
                if(read.get()>0){
                    buffer.flip();
                    String msg=String.valueOf(charset.decode(buffer));
                    System.out.println(msg);
                    buffer.clear();
                }else {
                    //如果read的结果小于等于0说明和服务器连接出现异常
                    System.out.println("服务器断开连接");
                    if(clientChannel!=null){
                        clientChannel.close();
                    }
                    System.exit(-1);
                }
            }
        } catch (IOException | InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }

    public void send(String msg) {
        if (msg.isEmpty())
            return;
        ByteBuffer buffer = charset.encode(msg);
        Future<Integer> write=clientChannel.write(buffer);
        try {
            //获取发送结果，如果get方法发生异常说明发送失败
            write.get();
        } catch (ExecutionException|InterruptedException e) {
            System.out.println("消息发送失败");
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new ChatClient("127.0.0.1",8888).start();
    }
}
```

## UserInputHandler

```java
public class UserInputHandler implements Runnable {
    ChatClient client;
    public UserInputHandler(ChatClient chatClient) {
        this.client=chatClient;
    }
    @Override
    public void run() {
        BufferedReader read=new BufferedReader(
                new InputStreamReader(System.in)
        );
        while (true){
            try {
                String input=read.readLine();
                client.send(input);
                if(input.equals("quit"))
                    break;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

运行测试:

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200417164605075-508533365.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200417164605075-508533365.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200417164620441-2122640649.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200417164620441-2122640649.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200417164642458-1627643597.png](https://gitee.com/littlefxc/oss/raw/master/images/1383122-20200417164642458-1627643597.png)