---
title: 手动搭建I/O网络通信框架1：Socket和ServerSocket入门实战，实现单聊
date: 2021-05-28 19:39:32
categories: Netty
tags: 
- netty
---

# 手动搭建I/O网络通信框架1：Socket和ServerSocket入门实战，实现单聊

> 转载自[https://www.cnblogs.com/lbhym/p/12673470.html](https://www.cnblogs.com/lbhym/p/12673470.html)

# 前言

这个基础项目会作为BIO、NIO、AIO的一个前提，后面会有数篇博客会基于这个小项目利用BIO、NIO、AIO进行改造升级。

简单的说一下io，了解的直接跳过看代码吧:IO常见的使用场景就是网络通信或读取文件等方面。IO流分为字节流和字符流。字节即Byte，包含八位二进制数，一个二进制数就是1bit，中文名称叫位。字符即一个字母或者一个汉字。一个字母由一个字节组成，而汉字根据编码不同由2个或者3个组成。Java.io包如下:详细的API可自行查阅资料

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200410142107890-242008210.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200410142107890-242008210.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200410142126015-790268014.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200410142126015-790268014.png)

**Socket定义**：套接字（socket）是一个抽象层，应用程序可以通过它发送或接收数据，可对其进行像对文件一样的打开、读写和关闭等操作。套接字允许应用程序将I/O插入到网络中，并与网络中的其他应用程序进行通信。网络套接字是IP地址与端口的组合。

**可以理解为两台机器或进程间进行网络通信的端点，这个端点包含IP地址和端口号。**

Socket和ServerSocket区别就如其名字一样，简单地说ServerSocket作用在服务端，用以监听客户端的请求。Socket作用在客户端和服务端，用以发送接收消息。但是就像上面说的，它们都要包含一个IP地址和端口号。

# **Socket和ServerSocket实战：**

首先创建一个最普通的Java项目。然后创建两个类，Server和Client。其代码和注释如下,仔细看下注释和代码，还是比较简单的

服务器只能为一个客户端服务，一旦监听到客户端的请求，就会一直被这个客户端占用。

```java
public class Client {
    public static void main(String[] args) {
        //这是服务端的IP和端口
        final String DEFAULT_SERVER_HOST = "127.0.0.1";
        final int DEFAULT_SERVER_PORT = 8888;
        //创建Socket
        try (Socket socket = new Socket(DEFAULT_SERVER_HOST, DEFAULT_SERVER_PORT)) {
            //接收消息
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream())
            );
            //发送消息
            BufferedWriter writer = new BufferedWriter(
                    new OutputStreamWriter(socket.getOutputStream())
            );
            //获取用户输入的消息
            BufferedReader userReader = new BufferedReader(
                    new InputStreamReader(System.in)
            );
            String msg = null;
            //循环的话客户端就可以一直输入消息，不然执行完try catch会自动释放资源，也就是断开连接
            while (true) {
                String input = userReader.readLine();
                //写入客户端要发送的消息。因为服务端用readLine获取消息，其以\n为终点，所以要在消息最后加上\n
                writer.write(input + "\n");
                writer.flush();
                msg = reader.readLine();
                System.out.println(msg);
                //如果客户端输入quit就可以跳出循环、断开连接了
                if(input.equals("quit")){
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

```java
public class Server {
    public static void main(String[] args) {
        final int DEFAULT_PORT = 8888;
        //创建ServerSocket监听8888端口
        try (ServerSocket serverSocket = new ServerSocket(DEFAULT_PORT)) {
            System.out.println("ServerSocket Start,The Port is:" + DEFAULT_PORT);
            while (true) {//不停地监听该端口
                //阻塞式的监听，如果没有客户端请求就一直停留在这里
                Socket socket = serverSocket.accept();
                System.out.println("Client[" + socket.getPort() + "]Online");
                //接收消息
                BufferedReader reader = new BufferedReader(
                        new InputStreamReader(socket.getInputStream())
                );
                //发送消息
                BufferedWriter writer = new BufferedWriter(
                        new OutputStreamWriter(socket.getOutputStream())
                );

                String msg = null;
                while ((msg = reader.readLine()) != null) {
                    System.out.println("Client[" + socket.getPort() + "]:" + msg);
                    //写入服务端要发送的消息
                    writer.write("Server:" + msg + "\n");
                    writer.flush();
                    //如果客户端的消息是quit代表他退出了，并跳出循环，不用再接收他的消息了。如果客户端再次连接就会重新上线
                    if (msg.equals("quit")) {
                        System.out.println("Client[" + socket.getPort() + "]:Offline");
                        break;
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

然后打开两个命令终端，通过javac编译后，一个运行Server代表服务器，一个运行Client代表客户端。

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200410144802569-1725038127.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200410144802569-1725038127.png)

![https://img2020.cnblogs.com/blog/1383122/202004/1383122-20200410144811290-2056832827.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1383122-20200410144811290-2056832827.png)

下一篇 [手动搭建I/O网络通信框架2：BIO编程模型实现群聊](%E6%89%8B%E5%8A%A8%E6%90%AD%E5%BB%BAI%20O%E7%BD%91%E7%BB%9C%E9%80%9A%E4%BF%A1%E6%A1%86%E6%9E%B62%EF%BC%9ABIO%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B%E5%AE%9E%E7%8E%B0%E7%BE%A4%E8%81%8A%202d2b7fd177844b1ab85c0276e7ae1e7b.md)  。