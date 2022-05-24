---
title: 手动搭建I/O网络通信框架3：NIO编程模型，升级改造聊天室
date: 2021-05-28 19:39:32
categories: Netty
tags: 
- netty
---


# 手动搭建I/O网络通信框架3：NIO编程模型，升级改造聊天室

> 转载自 [https://www.cnblogs.com/lbhym/p/12698309.html](https://www.cnblogs.com/lbhym/p/12698309.html)

# **手动搭建I/O网络通信框架3：NIO编程模型，升级改造聊天室**

在“手动搭建I/O网络通信框架2：BIO编程模型实现群聊”中用BIO编程模型，简单的实现了一个聊天室。但是其最大的问题在解释BIO时就已经说了：ServerSocket接收请求时（accept()方法）、InputStream、OutputStream（输入输出流的读和写）都是阻塞的。还有一个问题就是线程池，线程多了，服务器性能耗不起。线程少了，在聊天室这种场景下，让用户等待连接肯定不可取。今天要说到的NIO编程模型就很好的解决了这几个问题。有两个主要的替换地方：

1. 用Channel代替Stream。
2. 使用Selector监控多条Channel，起到类似线程池的作用，但是它只需一条线程。

既然要用NIO编程模型，那就要说说它的三个主要核心：Selector、Channel、Buffer。它们的关系是：一个Selector管理多个Channel，一个Channel可以往Buffer中写入和读取数据。Buffer名叫缓冲区，底层其实是一个数组，会提供一些方法往数组写入读取数据。

## **Buffer:**

不太了解Buffer的可以看看这个：[https://blog.csdn.net/czx2018/article/details/89502699](https://blog.csdn.net/czx2018/article/details/89502699)

**常用API：**

- allocate() - 初始化一块缓冲区
- put() - 向缓冲区写入数据
- get() - 向缓冲区读数据
- filp() - 将缓冲区的读写模式转换
- clear() - 这个并不是把缓冲区里的数据清除，而是利用后来写入的数据来覆盖原来写入的数据，以达到类似清除了老的数据的效果
- compact() - 从读数据切换到写模式，数据不会被清空，会将所有未读的数据copy到缓冲区头部，后续写数据不会覆盖，而是在这些数据之后写数据
- mark() - 对position做出标记，配合reset使用
- reset() - 将position置为标记值

*简单地说*：Buffer实质上是个数组，有两个关键的指针，一个position代表当前数据写入到哪了、一个limit代表限制。初始化时设置了数组长度，这limit就是数组的长度。如：设置`intBuffer.allocate(10)`，最大存储10个int数据，写入5五个数据后，需要读取数据了。用filp()转换读写模式后，limit=position，position=0。也就是说从0开始读，只能读到第五个。读完后这个缓冲区就需要clear()了，实际上并没有真的去清空数据，而是position和limit两个指针又回到了初始化的位置，接着又可以写入数据了，反正数组下标相同重新写入数据会覆盖，就没必要真的去清空了。

## **Channel:**

Channel(通道)主要用于传输数据，然后从Buffer中写入或读取。它们两个结合起来虽然和流有些相似，但主要有以下几点区别：　　

1. 流是单向的，可以发现Stream的输入流和输出流是独立的，它们只能输入或输出。而通道既可以读也可以写。　　
2. 通道本身不能存放数据，只能借助Buffer。　　
3. Channel支持异步。　　

Channel有如下三个常用的类：FileChannel、SocketChannel、ServerSocketChannel。从名字也可以看出区别，第一个是对文件数据的读写，后面两个则是针对Socket和ServerSocket，这里我们只是用后面两个。更详细的用法可以看：https://www.cnblogs.com/snailclimb/p/9086335.html，下面的代码中也会用到，会有详细的注释。

## **Selector**

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414150122707-1475260423.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414150122707-1475260423.png)

多个Channel可以注册到Selector，就可以直接通过一个Selector管理多个通道。Channel在不同的时间或者不同的事件下有不同的状态，Selector会通过轮询来达到监视的效果，如果查到Channel的状态正好是我们注册时声明的所要监视的状态，我们就可以查出这些通道，然后做相应的处理。这些状态如下：　　1.客户端的SocketChannel和服务器端建立连接，SocketChannel状态就是**Connect**。　　2.服务器端的ServerSocketChannel接收了客户端的请求，ServerSocketChannel状态就是**Accept**。　　3.当SocketChannel有数据可读，那么它们的状态就是**Read**。　　4.当我们需要向Channel中写数据时，那么它们的状态就是**Write**。　　

具体的使用见下面代码注释或看 [Java NIO之Selector（选择器）](Java%20NIO%E4%B9%8BSelector%EF%BC%88%E9%80%89%E6%8B%A9%E5%99%A8%EF%BC%89%20608ea119b60443d6a6233f32004d92ec.md) 

## **NIO编程模型**

NIO编程模型工作流程：　　

1. 首先会创建一个Selector，用来监视管理各个不同的Channel，也就是不同的客户端。相当于取代了原来BIO的线程池，但是它只需一个线程就可以处理多个Channel，没有了线程上下文切换带来的消耗，很好的优化了性能。　　
2. 创建一个ServerSocketChannel监听通信端口，并注册到Selector，让Seletor监视这个通道的Accept状态，也就是接收客户端请求的状态。
3. 此时客户端ClientA请求服务器，那么Selector就知道了有客户端请求进来。这时候我们可以得到客户端的SocketChannel，并为这个通道注册Read状态，也就是Selector会监听ClientA发来的消息。　　
4. 一旦接收到ClientA的消息，就会用其他客户端的SocketChannel的Write状态，向它们转发ClientA的消息。

上代码之前，还是先说说各个类的作用：

相比较BIO的代码，NIO的代码还少了一个类，那就是服务器端的工作线程类。没了线程池，自然也不需要一个单独的线程去服务客户端。客户端还是需要一个单独的线程去等待用户输入，因为用户随时都可能输入信息，这个没法预见，只能阻塞式的等待。

- ChatServer:服务器端的唯一的类，作用就是通过Selector监听Read和Accept事件，并针对这些事件的类型，进行不同的处理，如连接、转发。
- ChatClient:客户端，通过Selector监听Read和Connect事件。Read事件就是获取服务器转发的消息然后显示出来；Connect事件就是和服务器建立连接，建立成功后就可以发送消息。
- UserInputHandler:专门等待用户输入的线程，和BIO没区别。

### **ChatServer**

```java
public class ChatServer {
    //设置缓冲区的大小，这里设置为1024个字节
    private static final int BUFFER = 1024;

    //Channel都要配合缓冲区进行读写，所以这里创建一个读缓冲区和一个写缓冲区
    //allocate()静态方法就是设置缓存区大小的方法
    private ByteBuffer read_buffer = ByteBuffer.allocate(BUFFER);
    private ByteBuffer write_buffer = ByteBuffer.allocate(BUFFER);

    //为了监听端口更灵活，再不写死了，用一个构造函数设置需要监听的端口号
    private int port;

    public ChatServer(int port) {
        this.port = port;
    }

    private void start() {
        //创建ServerSocketChannel和Selector并打开
        try (ServerSocketChannel server = ServerSocketChannel.open(); Selector selector = Selector.open()) {
            //【重点,实现NIO编程模型的关键】configureBlocking设置ServerSocketChannel为非阻塞式调用,Channel默认的是阻塞的调用方式
            server.configureBlocking(false);
            //绑定监听端口,这里不是给ServerSocketChannel绑定，而是给ServerSocket绑定，socket()就是获取通道原生的ServerSocket或Socket
            server.socket().bind(new InetSocketAddress(port));

            //把server注册到Selector并监听Accept事件
            server.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("启动服务器，监听端口:" + port);

            while (true) {
                //select()会返回此时触发了多少个Selector监听的事件
                if(selector.select()>0) {
                    //获取这些已经触发的事件,selectedKeys()返回的是触发事件的所有信息
                    Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    //循环处理这些事件
                    for (SelectionKey key : selectionKeys) {
                        handles(key, selector);
                    }
                    //处理完后清空selectedKeys，避免重复处理
                    selectionKeys.clear();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //处理事件的方法
    private void handles(SelectionKey key, Selector selector) throws IOException {
        //当触发了Accept事件，也就是有客户端请求进来
        if (key.isAcceptable()) {
            //获取ServerSocketChannel
            ServerSocketChannel server = (ServerSocketChannel) key.channel();
            //然后通过accept()方法接收客户端的请求，这个方法会返回客户端的SocketChannel，这一步和原生的ServerSocket类似
            SocketChannel client = server.accept();
            client.configureBlocking(false);

            //把客户端的SocketChannel注册到Selector，并监听Read事件
            client.register(selector, SelectionKey.OP_READ);
            System.out.println("客户端[" + client.socket().getPort() + "]上线啦！");
        }
        //当触发了Read事件，也就是客户端发来了消息
        if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            //获取消息
            String msg = receive(client);
            System.out.println("客户端[" + client.socket().getPort() + "]:" + msg);
            //把消息转发给其他客户端
            sendMessage(client, msg, selector);
            //判断用户是否退出
            if (msg.equals("quit")) {
                //解除该事件的监听
                key.cancel();
                //更新Selector
                selector.wakeup();
                System.out.println("客户端[" + client.socket().getPort() + "]下线了！");
            }
        }
    }

    //编码方式设置为utf-8，下面字符和字符串互转时用得到
    private Charset charset = Charset.forName("UTF-8");

    //接收消息的方法
    private String receive(SocketChannel client) throws IOException {
        //用缓冲区之前先清空一下,避免之前的信息残留
        read_buffer.clear();
        //把通道里的信息读取到缓冲区，用while循环一直读取，直到读完所有消息。因为没有明确的类似\n这样的结尾，所以要一直读到没有字节为止
        while (client.read(read_buffer) > 0) ;
        //把消息读取到缓冲区后，需要转换buffer的读写状态，不明白的看看前面的Buffer的讲解
        read_buffer.flip();
        return String.valueOf(charset.decode(read_buffer));
    }

    //转发消息的方法
    private void sendMessage(SocketChannel client, String msg, Selector selector) throws IOException {
        msg = "客户端[" + client.socket().getPort() + "]:" + msg;
        //获取所有客户端,keys()与前面的selectedKeys不同，这个是获取所有已经注册的信息，而selectedKeys获取的是触发了的事件的信息
        for (SelectionKey key : selector.keys()) {
            //排除服务器和本客户端并且保证key是有效的，isValid()会判断Selector监听是否正常、对应的通道是保持连接的状态等
            if (!(key.channel() instanceof ServerSocketChannel) && !client.equals(key.channel()) && key.isValid()) {
                SocketChannel otherClient = (SocketChannel) key.channel();
                write_buffer.clear();
                write_buffer.put(charset.encode(msg));
                write_buffer.flip();
                //把消息写入到缓冲区后，再把缓冲区的内容写到客户端对应的通道中
                while (write_buffer.hasRemaining()) {
                    otherClient.write(write_buffer);
                }
            }
        }
    }

    public static void main(String[] args) {
        new ChatServer(8888).start();
    }
}
```

### **ChatClient**

```java
public class ChatClient {
    private static final int BUFFER = 1024;
    private ByteBuffer read_buffer = ByteBuffer.allocate(BUFFER);
    private ByteBuffer write_buffer = ByteBuffer.allocate(BUFFER);
    //声明成全局变量是为了方便下面一些工具方法的调用，就不用try with resource了
    private SocketChannel client;
    private Selector selector;

    private Charset charset = Charset.forName("UTF-8");

    private void start() {
        try  {
            client=SocketChannel.open();
            selector=Selector.open();
            client.configureBlocking(false);
            //注册channel，并监听SocketChannel的Connect事件
            client.register(selector, SelectionKey.OP_CONNECT);
            //请求服务器建立连接
            client.connect(new InetSocketAddress("127.0.0.1", 8888));
            //和服务器一样，不停的获取触发事件，并做相应的处理
            while (true) {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                for (SelectionKey key : selectionKeys) {
                    handle(key);
                }
                selectionKeys.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }catch (ClosedSelectorException e){
            //当用户输入quit时，在send()方法中，selector会被关闭，而在上面的无限while循环中，可能会使用到已经关闭了的selector。
            //所以这里捕捉一下异常，做正常退出处理就行了。不会对服务器造成影响
        }
    }

    private void handle(SelectionKey key) throws IOException {
        //当触发connect事件，也就是服务器和客户端建立连接
        if (key.isConnectable()) {
            SocketChannel client = (SocketChannel) key.channel();
            //finishConnect()返回true，说明和服务器已经建立连接。如果是false，说明还在连接中，还没完全连接完成
            if(client.finishConnect()){
                //新建一个新线程去等待用户输入
                new Thread(new UserInputHandler(this)).start();
            }
            //连接建立完成后，注册read事件，开始监听服务器转发的消息
            client.register(selector,SelectionKey.OP_READ);
        }
        //当触发read事件，也就是获取到服务器的转发消息
        if(key.isReadable()){
            SocketChannel client = (SocketChannel) key.channel();
            //获取消息
            String msg = receive(client);
            System.out.println(msg);
            //判断用户是否退出
            if (msg.equals("quit")) {
                //解除该事件的监听
                key.cancel();
                //更新Selector
                selector.wakeup();
            }
        }
    }
    //获取消息
    private String receive(SocketChannel client) throws IOException{
        read_buffer.clear();
        while (client.read(read_buffer)>0);
        read_buffer.flip();
        return String.valueOf(charset.decode(read_buffer));
    }

    //发送消息
    public void send(String msg) throws IOException{
        if(!msg.isEmpty()){
            write_buffer.clear();
            write_buffer.put(charset.encode(msg));
            write_buffer.flip();
            while (write_buffer.hasRemaining()){
                client.write(write_buffer);
            }
            if(msg.equals("quit")){
                selector.close();
            }
        }
    }

    public static void main(String[] args) {
        new ChatClient().start();
    }
}
```

### **UserInputHandler**

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

测试运行：之前用的是win10的终端运行的，以后直接用IDEA运行，方便些。不过一个类同时运行多个，以实现多个客户端的场景，需要先做以下设置

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152051636-927905062.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152051636-927905062.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152100412-1057042302.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152100412-1057042302.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152130678-206345317.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152130678-206345317.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152204091-2069873206.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152204091-2069873206.png)

设置完后，就可以同时运行两个ChatClient了，上图中得Unnamed就是第二个ChatClient，选中后点击右边运行按钮就行了。效果如下：

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152412715-1986061599.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152412715-1986061599.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152438453-1030565064.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152438453-1030565064.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200414152450583-1807958534.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200414152450583-1807958534.png)