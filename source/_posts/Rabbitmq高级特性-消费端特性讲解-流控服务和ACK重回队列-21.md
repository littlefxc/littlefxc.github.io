---
title: Rabbitmq高级特性-消费端特性讲解_流控服务和ACK重回队列(21)
date: 2021-01-26 21:06:04
categories: mq
tags: 
- mq
- rabbitmq
---

![rabbitmq_21_1](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_1.png)

![rabbitmq_21_2](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_2.png)

![rabbitmq_21_3](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_3.png)

![rabbitmq_21_4](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_4.png)

![rabbitmq_21_5](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_5.png)

![rabbitmq_21_6](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_6.png)

![rabbitmq_21_7](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_21_7.png)

## Sender

```java
public class Sender {

	
	public static void main(String[] args) throws Exception {
		
		//1 创建ConnectionFactory
		ConnectionFactory connectionFactory = new ConnectionFactory();
//		connectionFactory.setHost("192.168.11.71");
		connectionFactory.setHost("localhost");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		//2 创建Connection
		Connection connection = connectionFactory.newConnection();
		//3 创建Channel
		Channel channel = connection.createChannel();  
		//4 声明
		String queueName = "test001";  
        //参数: queue名字,是否持久化,独占的queue（仅供此连接）,不使用时是否自动删除, 其他参数
		channel.queueDeclare(queueName, true, false, false, null);
		
		for(int i = 0; i < 5;i++) {
			String msg = "Hello World RabbitMQ " + i;
			Map<String, Object> headers = new HashMap<String, Object>();
			headers.put("flag", i);
			AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
			.deliveryMode(2)
			.contentEncoding("UTF-8")
			.headers(headers).build();
			channel.basicPublish("", queueName , props , msg.getBytes()); 			
		}
	}
	
}
```

## Receiver

```java
public class Receiver {

	public static void main(String[] args) throws Exception {
		
		
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
//        connectionFactory.setHost("192.168.11.71");
        connectionFactory.setHost("localhost");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
        
        String queueName = "test001";  
        //durable 是否持久化消息
        channel.queueDeclare(queueName, true, false, false, null);  
        QueueingConsumer consumer = new QueueingConsumer(channel);
        
        //	参数：队列名称、是否自动ACK、Consumer
        channel.basicConsume(queueName, false, consumer);  
        //	循环获取消息  
        while(true){  
            //	获取消息，如果没有消息，这一步将会一直阻塞  
            Delivery delivery = consumer.nextDelivery();  
            String msg = new String(delivery.getBody());    
            System.out.println("收到消息：" + msg);  
            Thread.sleep(1000);
            
            if((Integer)delivery.getProperties().getHeaders().get("flag") == 0) {
            	//throw new RuntimeException("异常");
            	channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
            } else {
            	channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            }
        } 
	}
}
```

