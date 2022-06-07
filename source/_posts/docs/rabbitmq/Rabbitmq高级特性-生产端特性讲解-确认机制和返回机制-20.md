---
title: Rabbitmq高级特性-生产端特性讲解_确认机制和返回机制(20)
date: 2021-01-26 20:06:30
categories: mq
tags: 
- mq
- rabbitmq
---

[TOC]

![rabbitmq_20_1](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_1.png)

![rabbitmq_20_2](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_2.png)

![rabbitmq_20_3](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_3.png)

![rabbitmq_20_4](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_4.png)

![rabbitmq_20_5](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_5.png)

![rabbitmq_20_6](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_6.png)

![rabbitmq_20_7](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_20_7.png)

## Sender4ConfirmListener

```java
public class Sender4ConfirmListener {
	
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
		String exchangeName = "test_confirmlistener_exchange";
		String routingKey1 = "confirm.save";
		
    	//5 发送
		String msg = "Hello World RabbitMQ 4 Confirm Listener Message ...";
		
		channel.confirmSelect();
        channel.addConfirmListener(new ConfirmListener() {
			@Override
			public void handleNack(long deliveryTag, boolean multiple) throws IOException {
				System.err.println("------- error ---------");
			}
			@Override
			public void handleAck(long deliveryTag, boolean multiple) throws IOException {
				System.err.println("------- ok ---------");
			}
		});
        
		channel.basicPublish(exchangeName, routingKey1 , null , msg.getBytes()); 

 
	}
	
}
```



## Receiver4ConfirmListener

```java
public class Receiver4ConfirmListener {

	public static void main(String[] args) throws Exception {
		
		
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
        connectionFactory.setHost("192.168.11.71");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
		//4 声明
		String exchangeName = "test_confirmlistener_exchange";
		String exchangeType = "topic";
		String queueName = "test_confirmlistener_queue";
		//String routingKey = "user.*";
		String routingKey = "confirm.#";
		channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
        //durable 是否持久化消息
        QueueingConsumer consumer = new QueueingConsumer(channel);
        //参数：队列名称、是否自动ACK、Consumer
        channel.basicConsume(queueName, false, consumer);  
        //循环获取消息  
        while(true){  
            //获取消息，如果没有消息，这一步将会一直阻塞  
            Delivery delivery = consumer.nextDelivery();  
            String msg = new String(delivery.getBody());    
            System.out.println("收到消息：" + msg);  
            //手工签收消息
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        } 
	}
}
```

## Sender4ReturnListener

```java
public class Sender4ReturnListener {
	
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
		String exchangeName = "test_returnlistener_exchange";
		String routingKey1 = "abcd.save";
		String routingKey2 = "return.save";
		String routingKey3 = "return.delete.abc";
		
		//5 监听
    	channel.addReturnListener(new ReturnListener() {
			public void handleReturn(int replyCode,
						            String replyText,
						            String exchange,
						            String routingKey,
						            AMQP.BasicProperties properties,
						            byte[] body)
					throws IOException {
				System.out.println("**************handleReturn**********");
				System.out.println("replyCode: " + replyCode);
				System.out.println("replyText: " + replyText);
				System.out.println("exchange: " + exchange);
				System.out.println("routingKey: " + routingKey);
				System.out.println("body: " + new String(body));
			}
    	});
    	
    	//6 发送
		String msg = "Hello World RabbitMQ 4 Return Listener Message ...";
		
		boolean mandatory = true;
		channel.basicPublish(exchangeName, routingKey1 , mandatory, null , msg.getBytes()); 
//		channel.basicPublish(exchangeName, routingKey2 , null , msg.getBytes()); 	
///		channel.basicPublish(exchangeName, routingKey3 , null , msg.getBytes()); 
		
 
	}
	
}
```

## Receiver4ReturnListener

```java
public class Receiver4ReturnListener {

	public static void main(String[] args) throws Exception {
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
        connectionFactory.setHost("192.168.11.71");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
		//4 声明
		String exchangeName = "test_returnlistener_exchange";
		String exchangeType = "topic";
		String queueName = "test_returnlistener_queue";
		//String routingKey = "user.*";
		String routingKey = "return.#";
		channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
        //durable 是否持久化消息
        QueueingConsumer consumer = new QueueingConsumer(channel);
        //参数：队列名称、是否自动ACK、Consumer
        channel.basicConsume(queueName, true, consumer);  
        //循环获取消息  
        while(true){  
            //获取消息，如果没有消息，这一步将会一直阻塞  
            Delivery delivery = consumer.nextDelivery();  
            String msg = new String(delivery.getBody());    
            System.out.println("收到消息：" + msg);  
        } 
	}
}
```

