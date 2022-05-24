---
title: "RocketMQ事务消息示例"
date: 2022-04-07T17:14:16+08:00
categories: 分布式系统
tags:
- rocketmq
---

## 引言

分布式事务是一个复杂的问题，本文就基于 RocketMQ 来实现最终一种性方案的分布式事务的示例与测试。

 ## 概念

![RocketMQ事务消息示例](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/RocketMQ%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF%E7%A4%BA%E4%BE%8B.jpg)

整体的流程如上所示。

RocketMQ 事务消息的原理是基于两阶段提交和事务状态回查。

- 半消息：是指暂时不能被消费的消息，半消息实际上被放在主题名为 `RMQ_SYS_TRANS_HALF_TOPIC`下，当 producer 对半消息进行二次确认后，也就是上图的第 4 步后，consumer 才可以消费。

- 事务状态回查：如果上图的第 4 步，半消息提交因为种种原因（网络原因、producer崩溃）失败了，而导致 broker 不能收到 producer 的确认消息，那么 broker 就会定时扫描这些半消息，主动去确认。

  当然，这个定时机制也是可以配置的。

最重要的两个概念就介绍到这里啦，其它的就不啰嗦了。

业务流程：每增加一个订单，就增加相应的积分。

## 数据库

数据库有两个，一个包含订单表和事务日志表，另一个则只有订单积分表。

```sql
-- 本地业务
CREATE TABLE `orders` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `user_id` int unsigned NOT NULL,
  `goods_name` varchar(255) NOT NULL COMMENT '商品名',
  `total` int unsigned NOT NULL COMMENT '数量',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8;

-- 这张表专门用于事务状态回查
-- 当本地业务提交后，此表也插入一条记录，两者处于同一个事务中
-- 通过 RocketMQ事务ID 查询该表，如果返回记录，则证明本地事务已提交；如果未返回记录，则本地事务可能是未知状态或者是回滚状态。
CREATE TABLE `transaction_log` (
  `id` varchar(32) NOT NULL COMMENT '事务ID',
  `business` varchar(32) NOT NULL COMMENT '业务标识',
  `foreign_key` varchar(32) NOT NULL COMMENT '对应业务表中的主键',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

```sql
-- 远端业务
CREATE TABLE `order_credits` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `user_id` int unsigned NOT NULL COMMENT '用户ID',
  `order_id` int unsigned NOT NULL COMMENT '订单ID',
  `total` int unsigned NOT NULL COMMENT '积分数量',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8;
```

## 核心代码

### 事务管理工具类

```java
package com.fengxuechao.example.rocketmq;

import java.sql.*;

/**
 * 事务管理工具类
 *
 * @author fengxuechao
 * @date 2022/4/7
 */
public class TransactionUtil {

    private static final ThreadLocal<Connection> connections = new ThreadLocal<>();

    private TransactionUtil() {
    }

    /**
     * 开启事务, jdbcUrl 要记得修改
     */
    public static Connection startTransaction() {
        Connection connection = connections.get();
        if (connection == null) {
            try {
                connection = DriverManager.getConnection(
                        "jdbc:mysql://localhost:3306/rocketmq?serverTimezone=GMT%2B8",
                        "root",
                        "12345678");
                connection.setAutoCommit(false);
                connections.set(connection);
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        return connection;
    }

    public static int execute(String sql, Object... args) throws SQLException {
        PreparedStatement preparedStatement = createPreparedStatement(sql, args);
        return preparedStatement.executeUpdate();
    }

    public static ResultSet select(String sql, Object... args) throws SQLException {
        PreparedStatement preparedStatement = createPreparedStatement(sql, args);
        preparedStatement.execute();
        return preparedStatement.getResultSet();
    }

    private static PreparedStatement createPreparedStatement(String sql, Object[] args) throws SQLException {
        Connection connection = startTransaction();
        PreparedStatement preparedStatement = connection.prepareStatement(sql);
        if (args != null) {
            for (int i = 0; i < args.length; i++) {
                preparedStatement.setObject(i + 1, args[i]);
            }
        }
        return preparedStatement;
    }


    /**
     * 提交事务
     */
    public static void commit() {
        try (Connection connection = connections.get()) {
            connection.commit();
            connections.remove();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    /**
     * 回滚事务
     */
    public static void rollback() {
        try (Connection connection = connections.get()) {
            connection.rollback();
            connections.remove();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### 生产者业务代码

> 发送半消息。对应的是上图中的第 1 步。
>
> 注意点：如果发送事务消息，在这里我们的创建的实例必须是 `TransactionMQProducer`。

```java
package com.fengxuechao.example.rocketmq;

import com.alibaba.fastjson.JSON;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.TransactionListener;
import org.apache.rocketmq.client.producer.TransactionMQProducer;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

import java.io.UnsupportedEncodingException;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 消息事务生产者
 *
 * @author fengxuechao
 * @date 2022/4/6
 */
public class TransactionProducer {

    public static void main(String[] args) throws MQClientException, InterruptedException {

        // 生产者事务监听器
        TransactionListener transactionListener = new OrderTransactionListener();
        TransactionMQProducer producer = new TransactionMQProducer();
        ExecutorService executorService = new ThreadPoolExecutor(
                2, 5, 100,
                TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000),
                r -> new Thread(r, "client-transaction-msg-check-thread"));

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.setProducerGroup("producer_order_trans_group");
        producer.start();

        // 发送消息
        String topic = "transaction-topic";
        String tags = "trans-order";
        Order order = new Order();
        order.setId(1);
        order.setUserId(1);
        order.setGoodsName("小脆面");
        order.setTotal(2);
        String orderJson = JSON.toJSONString(order);
        try {
            byte[] orderBytes = orderJson.getBytes(RemotingHelper.DEFAULT_CHARSET);
            Message msg = new Message(topic, tags, "order", orderBytes);
            producer.sendMessageInTransaction(msg, null);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        producer.shutdown();
    }
}

```

>  - 半消息确认，执行本地事务。对应的是`executeLocalTransaction`这个方法，需要注意的是本地业务提交后，事务日志表也插入一条记录，两者处于同一个事务中。
>
>  - 回查事务状态。对应的是`checkLocalTransaction`这个方法。
>    - 在这里，我们通过事务ID查询`transaction_log`这张表，如果可以查询到结果，就提交事务消息；如果没有查询到，就返回未知状态。
>    - 如果返回未知状态，broker 会以1分钟的间隔时间不断回查，直至达到事务回查最大检测数，如果超过这个数字还未查询到事务状态，则回滚此消息。

```java
package com.fengxuechao.example.rocketmq;

import com.alibaba.fastjson.JSON;
import org.apache.rocketmq.client.producer.LocalTransactionState;
import org.apache.rocketmq.client.producer.TransactionListener;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageExt;

import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * @author fengxuechao
 * @date 2022/4/7
 */
public class OrderTransactionListener implements TransactionListener {
    /**
     * When send transactional prepare(half) message succeed, this method will be invoked to execute local transaction.
     *
     * @param msg Half(prepare) message
     * @param arg Custom business parameter
     * @return Transaction state
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // RocketMQ 半消息发送成功，开始执行本地事务
        System.out.println("执行本地事务");
        TransactionUtil.startTransaction();
        LocalTransactionState state;
        try {
            // 创建订单
            System.out.println("创建订单");
            String orderStr = new String(msg.getBody());
            Order order = JSON.parseObject(orderStr, Order.class);
            String sql = "insert into orders(id, user_id, goods_name, total) values(?, ?, ?, ?)";
            int executeUpdates = TransactionUtil.execute(sql, order.getId(), order.getUserId(),
                    order.getGoodsName(), order.getTotal());
            if (executeUpdates > 0) {
                // 写入本地事务日志
                System.out.println("写入本地事务日志");
                String logSql = "insert into transaction_log(id, business, foreign_key) values(?, ?, ?)";
                String business = msg.getKeys();
                TransactionUtil.execute(logSql, msg.getTransactionId(), business, order.getId());
            }
            TransactionUtil.commit();
            state = LocalTransactionState.COMMIT_MESSAGE;
        } catch (SQLException e) {
            TransactionUtil.rollback();
            state = LocalTransactionState.ROLLBACK_MESSAGE;
            System.out.println("本地事务异常，回滚");
            e.printStackTrace();
        }
        return state;
    }

    /**
     * When no response to prepare(half) message. broker will send check message to check the transaction status, and this
     * method will be invoked to get local transaction status.
     *
     * @param msg Check message
     * @return Transaction state
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // 回查本地事务
        System.out.printf("回查本地事务, transactionId = %s%n", msg.getTransactionId());
        TransactionUtil.startTransaction();
        String sql = "select id, business, foreign_key from transaction_log where id = ?";
        try (ResultSet transactionLog = TransactionUtil.select(sql, msg.getTransactionId())) {
            if (transactionLog == null) {
                return LocalTransactionState.UNKNOW;
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

### 消费者业务代码

> 这里是增加积分的阶段。
>
> - 需要注意的是幂等性消费，总的思路就是在执行业务前，必需确认该消息是否被处理过。可以使用 RocketMQ 事务消息的 ID，也可以使用订单ID。
>
> - 第二个需要注意的是消息一直不能成功消费。这个时候，我想到两种方式处理：
>   - 在代码中设置消息重试次数，然后发送邮件或其他方式通知业务方人工处理
>   - 或者等待消息达到最大重试次数，进入死信队列（主题：%DLQ% + 消费者组名称）。

```java
package com.fengxuechao.example.rocketmq;

import com.alibaba.fastjson.JSON;
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;

import java.nio.charset.StandardCharsets;
import java.sql.ResultSet;
import java.util.List;

/**
 * @author fengxuechao
 * @date 2022/4/7
 */
public class TransactionConsumer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_order_trans_group");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("transaction-topic", "trans-order");
        consumer.setMaxReconsumeTimes(3);
        TransactionUtil.startTransaction();
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                try {
                    for (MessageExt msg : msgs) {
                        // 多次消费消息处理仍然失败后，发送邮件，人工处理
                        if (msg.getReconsumeTimes() >= 3) {
                            // 发送邮件，人工处理
                            sendMail();
                        }

                        String orderStr = new String(msg.getBody(), StandardCharsets.UTF_8);
                        Order order = JSON.parseObject(orderStr, Order.class);
                        // 幂等性保持
                        String sql1 = "select * from order_credits where order_id = ?";
                        ResultSet rs = TransactionUtil.select(sql1, order.getId());
                        if (rs != null && rs.next()) {
                            System.out.println("积分已添加，订单已处理！");
                        } else {
                            // 增加积分
                            String sql2 = "insert into order_credits(user_id,order_id,total) values(?,?,?)";
                            TransactionUtil.execute(sql2, order.getUserId(), order.getId(), order.getTotal() * 2);
                            System.out.printf("订单（id=%s）添加积分%n", order.getId());
                            TransactionUtil.commit();
                        }
                    }
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                } catch (Exception e) {
                    TransactionUtil.rollback();
                    e.printStackTrace();
                }
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }

            private void sendMail() { }
        });

        consumer.start();

        System.out.println("Consumer Started.");
    }
}
```

总体上，我的思路就是这样，希望大家一起讨论学习。

## 链接

- [GIthub 官方 消息事务样例](https://github.com/apache/rocketmq/blob/master/docs/cn/RocketMQ_Example.md#1%E5%88%9B%E5%BB%BA%E4%BA%8B%E5%8A%A1%E6%80%A7%E7%94%9F%E4%BA%A7%E8%80%85)

  除了消息事务，还有其他消息发送的样例

- [自定义脚本启动搭建的RocketMQ伪集群](https://blog.csdn.net/Little_fxc/article/details/123923938?spm=1001.2014.3001.5501)

  简单的 RocketMQ 的部署方式讲解，及一键启动脚本，方便测试。

  
