---
title: springboot使用rabbitmq
date: 2019-09-12 14:03:42
categories: amqp
tags:
- rabbitmq
- 消息队列
---

## Message Broker与AMQP简介

Message Broker是一种消息验证、传输、路由的架构模式，其设计目标主要应用于下面这些场景：

- 消息路由到一个或多个目的地
- 消息转化为其他的表现方式
- 执行消息的聚集、消息的分解，并将结果发送到他们的目的地，然后重新组合相应返回给消息用户
- 调用Web服务来检索数据
- 响应事件或错误
- 使用发布-订阅模式来提供内容或基于主题的消息路由

AMQP是Advanced Message Queuing Protocol的简称，它是一个面向消息中间件的开放式标准应用层协议。AMQP定义了这些特性：

- 消息方向
- 消息队列
- 消息路由（包括：点到点和发布-订阅模式）
- 可靠性
- 安全性

### RabbitMQ

RabbitMQ就是以AMQP协议实现的一种中间件产品，它可以支持多种操作系统，多种编程语言，几乎可以覆盖所有主流的企业级技术平台。

#### ubuntu 安装

在Ubuntu中，我们可以使用APT仓库来进行安装

1. 安装Erlang，执行：`apt-get install erlang`
2. 执行下面的命令，新增APT仓库到 `/etc/apt/sources.list.`

    ```sh
    echo 'deb http://www.rabbitmq.com/debian/ testing main' | sudo tee /etc/apt/sources.list.d/rabbitmq.list
    ```

3. 更新APT仓库的package list，执行 `sudo apt-get update` 命令
4. 安装Rabbit Server，执行 `sudo apt-get install rabbitmq-server` 命令
5. 开启Web管理插件

    ```sh
    rabbitmq-plugins enable rabbitmq_management
    ```

#### docker 安装

1. 查找镜像

   ```sh
   docker search rabbitmq
   ```

2. 拉取镜像

    ```sh
    docker pull rabbitmq
    ```

3. 启动镜像

    ```sh
    docker run -d -p 15672:15672 -p 5672:5672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin --name rabbitmq --hostname=my_rabbitmq rabbitmq:latest
    ```

    参数解释：
    - --hostname：指定容器主机名称
    - --name:指定容器名称
    - -p:将mq端口号映射到本地
    - 15672 ：表示 RabbitMQ 控制台端口号，可以在浏览器中通过控制台来执行 RabbitMQ 的相关操作。
    - 5672 : 表示 RabbitMQ 所监听的 TCP 端口号，应用程序可通过该端口与 RabbitMQ 建立 TCP 连接，完成后续的异步消息通信
    - RABBITMQ_DEFAULT_USER：用于设置登陆控制台的用户名，设置为 `admin`, 默认是 `guest`
    - RABBITMQ_DEFAULT_PASS：用于设置登陆控制台的密码，设置为 `admin`, 默认是 `guest`

    容器启动成功后，可以在浏览器输入地址：[http://localhost:15672/](http://localhost:15672/)

    ps：RabbitMQ出于安全的考虑，默认是只能访问localhost:15762访问的，如果想用其他ip，是需要自己配置的。

#### WEB 界面

![rabbitmqserver_web.png](/images/rabbitmqserver_web.png)

### rabbitmq 介绍

### spring boot 集成 RabbitMQ

1. maven 依赖

    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    ```

2. 配置文件

    ```yaml
    spring:
      application:
        name: spring-boot-rabbitmq
      rabbitmq:
        host: localhost
        port: 5672
        username: guest
        password: guest
    ```

3. 简单发送字符串

   1. 定义队列

    ```java
    import org.springframework.amqp.core.Queue;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;

    /**
     * @author fengxuechao
     * @version 0.1
     * @date 2019/9/12
     */
    @Configuration
    public class RabbitConfig {

        @Bean
        public Queue queueHello() {
            return new Queue("hello");
        }

        @Bean
        public Queue queueObject() {
            return new Queue("object");
        }
    }
    ```

    2. 定义发送者

    ```java
    import org.springframework.amqp.core.AmqpTemplate;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Component;

    import java.time.LocalDateTime;
    import java.time.format.DateTimeFormatter;

    /**
     * @author fengxuechao
     * @version 0.1
     * @date 2019/9/12
     */
    @Component
    public class HelloSender {

        @Autowired
        private AmqpTemplate rabbitTemplate;

        public void send(int i) {
            String context = "hello " + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS")) + ":" + i;
            System.out.println("Sender : " + context);
            this.rabbitTemplate.convertAndSend("hello", context);
        }

    }
    ```

    3. 定义接收者

    ```java
    import org.springframework.amqp.rabbit.annotation.RabbitHandler;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.stereotype.Component;

    /**
     * @author fengxuechao
     * @version 0.1
     * @date 2019/9/12
     */
    @Component
    @RabbitListener(queues = "hello")
    public class HelloReceiver2 {

        @RabbitHandler
        public void process(String hello) {
            System.out.println("Receiver 2 : " + hello);
        }

    }
    ```

    4. 测试

    ```java
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.annotation.Repeat;
    import org.springframework.test.context.junit4.SpringRunner;

    import static org.junit.Assert.*;

    /**
     * @author fengxuechao
     * @version 0.1
     * @date 2019/9/12
     */
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class HelloSenderTest {

        @Autowired
        private HelloSender sender;

        @Test
        public void send() {
            for (int i=0;i<10;i++){
                sender.send(i);
            }
        }
    }
    ```

    控制台打印结果

    ```log
    Receiver 2 : hello 2019-09-12 17:27:42.744:0
    Receiver 2 : hello 2019-09-12 17:27:42.744:1
    Receiver 2 : hello 2019-09-12 17:27:42.744:2
    Receiver 2 : hello 2019-09-12 17:27:42.744:3
    Receiver 2 : hello 2019-09-12 17:27:42.744:4
    Receiver 2 : hello 2019-09-12 17:27:42.744:5
    Receiver 2 : hello 2019-09-12 17:27:42.744:6
    Receiver 2 : hello 2019-09-12 17:27:42.744:7
    Receiver 2 : hello 2019-09-12 17:27:42.744:8
    Receiver 2 : hello 2019-09-12 17:27:42.744:9
    ```

4. 发送对象

    Spring Boot 完美的支持对象的发送和接收，不需要额外的配置。

    ```java
    @Bean
    public Queue queueObject() {
        return new Queue("object");
    }

    @Component
    @RabbitListener(queues = "object")
    public class ObjectReceiver {

        @RabbitHandler
        public void process(User user) {
            System.out.println("Receiver object : " + user);
        }

    }

    @Component
    public class ObjectSender {

        @Autowired
        private AmqpTemplate rabbitTemplate;

        public void send(User user) {
            System.out.println("Sender object: " + user.toString());
            this.rabbitTemplate.convertAndSend("object", user);
        }

    }
    ```

    结果如下：

    ```log
    Sender object: User{name='neo', pass='123456'}
    Receiver object : User{name='neo', pass='123456'}
    ```

5. 
