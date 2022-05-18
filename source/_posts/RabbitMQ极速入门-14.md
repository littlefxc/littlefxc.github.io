---
title: RabbitMQ极速入门(14)
date: 2021-01-25 21:46:34
categories: mq
tags: 
- mq
- rabbitmq
---

![https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_14_1.png](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_14_1.png)

![https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_14_2.png](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_14_2.png)

## 生产者

```java
public class Sender {

	
	public static void main(String[] args) throws Exception {
		
		//	1 创建ConnectionFactory
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("192.168.11.71");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		//	2 创建Connection
		Connection connection = connectionFactory.newConnection();
		//	3 创建Channel
		Channel channel = connection.createChannel();  
		//	4 声明
		String queueName = "test001";  
        //	参数: queue名字,是否持久化,独占的queue（仅供此连接）,不使用时是否自动删除, 其他参数
		channel.queueDeclare(queueName, false, false, false, null);
		
		Map<String, Object> headers = new HashMap<String, Object>();
		
		AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
		.deliveryMode(2)
		.contentEncoding("UTF-8")
		.headers(headers).build();
		
		
		for(int i = 0; i < 5;i++) {
			String msg = "Hello World RabbitMQ " + i;
			channel.basicPublish("", queueName , props , msg.getBytes()); 			
		}
	}
	
}
```

## 消费者

```java
public class Receiver {

	public static void main(String[] args) throws Exception {
		
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
        connectionFactory.setHost("192.168.11.71");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
        
        String queueName = "test001";  
        //	durable 是否持久化消息
        channel.queueDeclare(queueName, false, false, false, null);  
        QueueingConsumer consumer = new QueueingConsumer(channel);
        //	参数：队列名称、是否自动ACK、Consumer
        channel.basicConsume(queueName, true, consumer);  
        //	循环获取消息  
        while(true){  
            //	获取消息，如果没有消息，这一步将会一直阻塞  
            Delivery delivery = consumer.nextDelivery();  
            String msg = new String(delivery.getBody());    
            System.out.println("收到消息：" + msg);  
        } 
	}
}

```

