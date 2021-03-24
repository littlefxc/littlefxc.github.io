---
title: rabbitmq整合springboot2.x(23)
date: 2021-01-27 20:18:21
categories: mq
tags: 
- mq
- rabbitmq
- spring-boot
---

![rabbitmq_23_3](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_23_3.png)

![rabbitmq_23_4](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_23_4.png)

![rabbitmq_23_5](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_23_5.png)

![rabbitmq_23_6](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_23_6.png)

![rabbitmq_23_8](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_23_8.png)

## 生产者关键代码

### application.properties

```java
server.servlet.context-path=/
server.port=8001

spring.rabbitmq.addresses=192.168.11.71:5672,192.168.11.72:5672,192.168.11.71:5673
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
spring.rabbitmq.connection-timeout=15000

##	使用启用消息确认模式
spring.rabbitmq.publisher-confirms=true

## 	设置return消息模式，注意要和mandatory一起去配合使用
##spring.rabbitmq.publisher-returns=true
##spring.rabbitmq.template.mandatory=true

spring.application.name=rabbit-producer
spring.http.encoding.charset=UTF-8
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
spring.jackson.default-property-inclusion=NON_NULL
```

### RabbitSender

```java
import java.util.Map;
import java.util.UUID;

import org.springframework.amqp.AmqpException;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.core.RabbitTemplate.ConfirmCallback;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;


@Component
public class RabbitSender {

	@Autowired
	private RabbitTemplate rabbitTemplate;
	
	/**
	 * 	这里就是确认消息的回调监听接口，用于确认消息是否被broker所收到
	 */
	final ConfirmCallback confirmCallback = new RabbitTemplate.ConfirmCallback() {
		/**
		 * 	@param CorrelationData 作为一个唯一的标识
		 * 	@param ack broker 是否落盘成功 
		 * 	@param cause 失败的一些异常信息
		 */
		@Override
		public void confirm(CorrelationData correlationData, boolean ack, String cause) {
			System.err.println("消息ACK结果:" + ack + ", correlationData: " + correlationData.getId());
		}
	};
	
	/**
	 * 	对外发送消息的方法
	 * @param message 	具体的消息内容
	 * @param properties	额外的附加属性
	 * @throws Exception
	 */
	public void send(Object message, Map<String, Object> properties) throws Exception {
		
		MessageHeaders mhs = new MessageHeaders(properties);
		Message<?> msg = MessageBuilder.createMessage(message, mhs);
		
		rabbitTemplate.setConfirmCallback(confirmCallback);
		
		// 	指定业务唯一的iD
		CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
		
		MessagePostProcessor mpp = new MessagePostProcessor() {
			
			@Override
			public org.springframework.amqp.core.Message postProcessMessage(org.springframework.amqp.core.Message message)
					throws AmqpException {
				System.err.println("---> post to do: " + message);
				return message;
			}
		};
		
		rabbitTemplate.convertAndSend("exchange-1",
				"springboot.rabbit", 
				msg, mpp, correlationData);
		
	}
}
```

## 消费者关键代码

### application.properties

```java
server.servlet.context-path=/
server.port=8002

spring.rabbitmq.addresses=192.168.11.71:5672,192.168.11.72:5672,192.168.11.71:5673
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
spring.rabbitmq.connection-timeout=15000

## 	表示消费者消费成功消息以后需要手工的进行签收(ack)，默认为auto
spring.rabbitmq.listener.simple.acknowledge-mode=manual
spring.rabbitmq.listener.simple.concurrency=5
spring.rabbitmq.listener.simple.max-concurrency=10
spring.rabbitmq.listener.simple.prefetch=1


##	作业：
##	最好不要在代码里写死配置信息，尽量使用这种方式也就是配置文件的方式
##	在代码里使用 	${}	方式进行设置配置: ${spring.rabbitmq.listener.order.exchange.name}
spring.rabbitmq.listener.order.exchange.name=order-exchange
spring.rabbitmq.listener.order.exchange.durable=true
spring.rabbitmq.listener.order.exchange.type=topic
spring.rabbitmq.listener.order.exchange.key=order.*

spring.application.name=rabbit-producer
spring.http.encoding.charset=UTF-8
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
spring.jackson.default-property-inclusion=NON_NULL
```

### RabbitReceive

```java
@Component
public class RabbitReceive {
	
	/**
	 * 	组合使用监听
	 * 	@RabbitListener @QueueBinding @Queue @Exchange
	 * @param message
	 * @param channel
	 * @throws Exception
	 */
	@RabbitListener(bindings = @QueueBinding(
					value = @Queue(value = "queue-1", durable = "true"),
					exchange = @Exchange(name = "exchange-1",
					durable = "true",
					type = "topic",
					ignoreDeclarationExceptions = "true"),
					key = "springboot.*"
				)
			)
	@RabbitHandler
	public void onMessage(Message message, Channel channel) throws Exception {
		//	1. 收到消息以后进行业务端消费处理
		System.err.println("-----------------------");
		System.err.println("消费消息:" + message.getPayload());

		//  2. 处理成功之后 获取deliveryTag 并进行手工的ACK操作, 因为我们配置文件里配置的是 手工签收
		//	spring.rabbitmq.listener.simple.acknowledge-mode=manual
		Long deliveryTag = (Long)message.getHeaders().get(AmqpHeaders.DELIVERY_TAG);
		channel.basicAck(deliveryTag, false);
	}
}
```

