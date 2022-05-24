---
title: Rabbitmq高级特性-TTL消息与死信队列详解(22)
date: 2021-01-26 21:39:44
categories: mq
tags: 
- mq
- rabbitmq
---

![rabbitmq_22_1](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_1.png)

![rabbitmq_22_2](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_2.png)

![rabbitmq_22_3](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_3.png)

![rabbitmq_22_4](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_4.png)

![rabbitmq_22_5](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_5.png)

![rabbitmq_22_7](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_22_7.png)

## Sender4DLXExchange

```java
public class Sender4DLXExchange {

	
	public static void main(String[] args) throws Exception {
		
		//1 创建ConnectionFactory
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("192.168.11.71");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		//2 创建Connection
		Connection connection = connectionFactory.newConnection();
		//3 创建Channel
		Channel channel = connection.createChannel();  
		//4 声明
		String exchangeName = "test_dlx_exchange";
		String routingKey = "group.bfxy";
		//5 发送
		
		Map<String, Object> headers = new HashMap<String, Object>();
		
		AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
		.deliveryMode(2)
		.contentEncoding("UTF-8")
		//	TTL
		.expiration("6000")
		.headers(headers).build();
		
		String msg = "Hello World RabbitMQ 4 DLX Exchange Message ... ";
		channel.basicPublish(exchangeName, routingKey , props , msg.getBytes()); 		
		
	}
	
}
```

## Receiver4DLXtExchange

```java
public class Receiver4DLXtExchange {

	public static void main(String[] args) throws Exception {
		
		
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
        connectionFactory.setHost("192.168.11.71");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
		//4 声明正常的 exchange queue 路由规则
		String queueName = "test_dlx_queue";
		String exchangeName = "test_dlx_exchange";
		String exchangeType = "topic";
		String routingKey = "group.*";
		//	声明 exchange
		channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
		
		
		//	注意在这里要加一个特殊的属性arguments: x-dead-letter-exchange
		Map<String, Object> arguments = new HashMap<String, Object>();
		arguments.put("x-dead-letter-exchange", "dlx.exchange");
		//arguments.put("x-dead-letter-routing-key", "dlx.*");
		//arguments.put("x-message-ttl", 6000);
		channel.queueDeclare(queueName, false, false, false, arguments);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		
		//dlx declare:
		channel.exchangeDeclare("dlx.exchange", exchangeType, true, false, false, null);
		channel.queueDeclare("dlx.queue", false, false, false, null);
		channel.queueBind("dlx.queue", "dlx.exchange", "#");
		
		
        //	durable 是否持久化消息
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

