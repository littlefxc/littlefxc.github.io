---
title: Kafka入门(1)
date: 2021-02-02 23:24:14
categories: mq
tags: 
- mq
- kafka
---

[TOC]

# Kafka 核心概念详解

## Kafka(MQ) 的应用场景

### Kafka(MQ)之异步化、服务解耦、削峰填谷

- 异步化
  
  ![kafka_1_1](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_1.png)

- 服务解耦、削峰填谷
  
  ![kafka_1_2](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_2.png)
  

<!-- more -->

### Kafka 海量日志收集

![kafka_1_3](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_3.png)

![image-20210203211950113](https://gitee.com/littlefxc/oss/raw/master/images/image-20210203211950113.png)

- Kafka 之数据同步应用

  ![kafka_1_4](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_4.png)

- Kafka 之实时计算
  
  ![kafka_1_5](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_5.png)

## Kafka 基本概念

### 集群架构概念

![kafka_1_6](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_6.png)

### Topic、Partition

![kafka_1_7](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_7.png)

### 副本(replica)

![kafka_1_8](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_8.png)

### ISR详解(In Sync Replicas)

![kafka_1_9](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_9.png)

上图表示拉取及时的情况

![kafka_1_10](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_10.png)

上图表示拉取滞后的情况。

<font color=red>**PS: 当Kafka集群中的 leader 挂了之后，Kafka集群会重新选举leader，这是只有在 ISR 集合里面的Kafka才会被选举成为leader。**</font>



- HW: High Watermark， 高水位线，消费者只能最多拉取到高水位线的消息

- LEO: Log End Offset，日志文件的最后一条记录的 offset(偏移量)

- ISR 集合与 HW 和 LEO 直接存在着密不可分的关系

![kafka_1_11](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_11.png)

![kafka_1_12](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_12.png)

![kafka_1_13](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_13.png)

上图右边的图形表示数据传入到leader节点，但还没有同步到follower节点上

![kafka_1_14](https://gitee.com/littlefxc/oss/raw/master/images/kafka_1_14.png)

上图HW移动了一格，表示 follower1 节点和follower2 节点都同步了第3条数据，而第4条数据因为follower2节点没有同步到，Kafka消费者就消费不了第4条数据。

## Kafka 环境搭建

[zookeeper 集群搭建](https://blog.csdn.net/Little_fxc/article/details/108626224?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522161236585916780269848148%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=161236585916780269848148&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v1~rank_blog_v1-1-108626224.pc_v1_rank_blog_v1&utm_term=zookeeper&spm=1018.2226.3001.4450)

[Kafka 集群搭建](https://www.cnblogs.com/luzhanshi/p/13369834.html)

[Kafka Manager - Kafka集群管理工具](https://www.cnblogs.com/pzb-shadow/p/13030365.html)
[kafka 命令行工具常用命令行操作](https://blog.csdn.net/asd136912/article/details/103735037)

## Kafka 极速入门

### 构建生产者步骤

1. 配置生产者参数属性和创建生产者对象
2. 构建消息：ProducerRecord
3. 发送消息
4. 关闭生产者

### 构建消费者步骤

1. 配置消费者参数属性和创建消费者对象
2. 订阅主题
3. 拉取消息并进行消费处理
4. 提交消费偏移量，关闭消费者

### 代码实现

#### 配置类

```java
/**
 * kafka配置类
 */
@Configuration
public class KafkaConfig {

    @Bean
    public KafkaProducer<String, String> producerRecord() {

        Properties properties = new Properties();
        // 配置kafka集群地址，不用将全部机器都写上，zk会自动发现全部的kafka broke
        properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093");
        // 设置发送消息的应答方式
        properties.setProperty(ProducerConfig.ACKS_CONFIG, "all");
        // 重试次数
        properties.setProperty(ProducerConfig.RETRIES_CONFIG, "3");
        // 重试间隔时间
        properties.setProperty(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, "100");
        // 一批次发送的消息大小 16KB
        properties.setProperty(ProducerConfig.BATCH_SIZE_CONFIG, "16348");
        // 一个批次等待时间,10ms
        properties.setProperty(ProducerConfig.LINGER_MS_CONFIG, "10");
        // RecordAccumulator 缓冲区大小  32M，如果缓冲区满了会阻塞发送端
        properties.setProperty(ProducerConfig.BUFFER_MEMORY_CONFIG, "33554432");
        // 配置拦截器, 多个逗号隔开
        properties.setProperty(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, "com.xiaolyuh.interceptor.TraceInterceptor");

        Serializer<String> keySerializer = new StringSerializer();
        Serializer<String> valueSerializer = new StringSerializer();

        return new KafkaProducer<>(properties, keySerializer, valueSerializer);
    }

}


```

#### 生产者

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringBootStudentKafkaApplicationTests {

    @Autowired
    private KafkaProducer<String, String> kafkaProducer;

    @Test
    public void testSyncKafkaSend() throws Exception {
        // 同步发送测试
        for (int i = 0; i < 100; i++) {
            ProducerRecord<String, String> producerRecord = new ProducerRecord<>("test_cluster_topic", "key-" + i, "value-" + i);
            // 同步发送，这里我们还可以指定发送到那个分区，还可以添加header
            kafkaProducer.send(producerRecord, new KafkaCallback<>(producerRecord)).get(50, TimeUnit.MINUTES);
        }

        System.out.println("ThreadName::" + Thread.currentThread().getName());
    }

    @Test
    public void testAsyncKafkaSend() {
        // 异步发送测试
        for (int i = 0; i < 100; i++) {
            ProducerRecord<String, String> producerRecord = new ProducerRecord<>("test_cluster_topic2", "key-" + i, "value-" + i);
            // 异步发送，这里我们还可以指定发送到那个分区，还可以添加header
            kafkaProducer.send(producerRecord, new KafkaCallback<>(producerRecord));
        }

        System.out.println("ThreadName::" + Thread.currentThread().getName());
        // 记得刷新，否则消息有可能没有发出去
        kafkaProducer.flush();
    }
}

/**
 * 异步回调函数，该方法会在 Producer 收到 ack 时调用，当Exception不为空表示发送消息失败。
 *
 * @param <K>
 * @param <V>
 */
class KafkaCallback<K, V> implements Callback {
    private final ProducerRecord<K, V> producerRecord;

    public KafkaCallback(ProducerRecord<K, V> producerRecord) {
        this.producerRecord = producerRecord;
    }

    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        System.out.println("ThreadName::" + Thread.currentThread().getName());
        if (Objects.isNull(exception)) {
            System.out.println(metadata.partition() + "-" + metadata.offset() + ":::" + producerRecord.key() + "=" + producerRecord.value());
        }

        if (Objects.nonNull(exception)) {
            // TODO  告警，消息落库从发
        }
    }
}
```

#### 消费者

Kafka中的消息消费是一个不断轮询的过程，消费者所要做的就是重复地调用`poll()`方法，而`poll()`方法返回的是所订阅的主题（分区）上的一组消息。

```java
@Component
public class KafkaConsumerDemo {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(1, 10,
            0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(1));

    @PostConstruct
    public void startConsumer() {
        executor.submit(() -> {
            Properties properties = new Properties();
            properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093");
            
            // 非常重要的属性配置：与我们的消费者订阅组有关系
            properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "groupId");
            // 消费者提交 offset：自动提交 & 手工提交，默认是自动提交
            properties.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
            // 请求超时时间
            properties.setProperty(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG, "60000");
            // 序列化
            Deserializer<String> keyDeserializer = new StringDeserializer();
            Deserializer<String> valueDeserializer = new StringDeserializer();
            // 创建消费者对象
            KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties, keyDeserializer, valueDeserializer);
            // 订阅感兴趣的主题
            consumer.subscribe(Arrays.asList("test_cluster_topic"));

            // KafkaConsumer的assignment（）方法来判定是否分配到了相应的分区，如果为空表示没有分配到分区
            Set<TopicPartition> assignment = consumer.assignment();
            while (assignment.isEmpty()) {
                // 阻塞1秒
                consumer.poll(1000);
                assignment = consumer.assignment();
            }

            // KafkaConsumer 分配到了分区，开始消费
            while (true) {
                // 拉取记录，如果没有记录则柱塞1000ms。
                ConsumerRecords<String, String> records = consumer.poll(1000);
                for (ConsumerRecord<String, String> record : records) {
                    String traceId = new String(record.headers().lastHeader("traceId").value());
                    System.out.printf("traceId = %s, offset = %d, key = %s, value = %s%n", traceId, record.offset(), record.key(), record.value());
                }

                // 异步确认提交
                consumer.commitAsync((offsets, exception) -> {
                    if (Objects.isNull(exception)) {
                        // TODO 告警、落盘、重试
                    }
                });
            }
        });

    }
}
```

#### 拦截器

```java
/**
 * 链路ID
 */
public class TraceInterceptor implements ProducerInterceptor<String, String> {
    private int errorCounter = 0;
    private int successCounter = 0;

    /**
     * 最先调用，读取配置信息，只调用一次
     */
    @Override
    public void configure(Map<String, ?> configs) {
        System.out.println(JSON.toJSONString(configs));
    }

    /**
     * 它运行在用户主线程中，在消息序列化和计算分区之前调用，这里最好不小修改topic 和分区参数，否则会出一些奇怪的现象。
     *
     * @param record
     * @return
     */
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {

        Headers headers = new RecordHeaders();
        headers.add("traceId", UUID.randomUUID().toString().getBytes(Charset.forName("UTF8")));
        // 修改消息
        return new ProducerRecord<>(record.topic(), record.partition(), record.key(), record.value(), headers);
    }

    /**
     * 该方法会在消息从 RecordAccumulator 成功发送到 Kafka Broker 之后，或者在发送过程 中失败时调用。
     * 并且通常都是在 producer 回调逻辑触发之前调用。
     * onAcknowledgement 运行在 producer 的 IO 线程中，因此不要在该方法中放入很重的逻辑，否则会拖慢 producer 的消息 发送效率。
     *
     * @param metadata
     * @param exception
     */
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (Objects.isNull(exception)) {
            // TODO  出错了
        }
    }

    /**
     * 关闭 interceptor，主要用于执行一些资源清理工作，只调用一次
     */
    @Override
    public void close() {
        System.out.println("==========close============");
    }
}
```

## Kafka 基本配置参数讲解

- 配置文件: `$KAFKA_HOME/config/server.properties`

  - zookeeper.connect

    CS格式参数，可以指定值为zk1:2181,zk2:2181,zk3:2181，不同Kafka集群可以指定：zk1:2181,zk2:2181,zk3:2181/kafka1，chroot只需要写一次

  - listeners

    设置内网访问Kafka服务的监听器

  - broker.id

    每个broker都可以用一个唯一的非负整数id进行标识；这个id可以作为broker的“名字”，并且它的存在使得broker无须混淆consumers就可以迁移到不同的host/port上。你可以选择任意你喜欢的数字作为id，只要id是唯一的即可。

  - log.dir 和 log.dirs

    kafka存放数据的路径。这个路径并不是唯一的，可以是多个，路径之间只需要使用逗号分隔即可；每当创建新partition时，都会选择在包含最少partitions的路径下进行。

  - message.max.bytes

    server可以接收的消息最大尺寸。重要的是，consumer和producer有关这个属性的设置必须同步，否则producer发布的消息对consumer来说太大。
  
- [详细配置参数](https://littlefxc.gitee.io/blog/passages/KAFKA-配置参数详解/)

## Kafka 之生产者

### 发送消息：ProducerRecord

```java
public class ProducerRecord<K, V> {
    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
    // ...
}
```

PS: 一条消息会通过 Key 去计算出来实际的 partition，按照 partitiion 去存储的。

### 必要的参数配置项

- bootstrap.servers：逗号分隔符，多个地址，防止单点故障

- key.serializer, value.serializer：kafka实际发送的是二进制的内容，所以必须序列化

- client.id：kafka 对应生产者的ID。如果不设置，Kafka 内部会自动生成一个非空字符串

- 简化的配置Key: ProducerConfig

- KafkaProducer 是线程安全的（kafka消费者不是线程安全的）

### 发送消息的3种方法

![image-20210225221350755](https://gitee.com/littlefxc/oss/raw/master/images/image-20210225221350755.png)

Kafka 发送消息提供了 3 种方法：

- sendOffsetsToTransaction: 事务相关
- send(ProducerRecord<K,V>):Future<RecordMetadata>：异步，但是使用 Future.get()方法相当于同步
- send(ProducerRecord<K,V>, Callback):Future<RecordMetadata>：异步，返回值会放到 Callback 回调函数里

### KafkaProducer 消息发送重试机制

- retries 参数
- 可重试异常(例如：网络抖动) & 不可重试异常(例如：磁盘满了、消息体积太大)



## Kafka 之生产者重要参数详解

- acks: 指定发送消息后，Broker端至少有多少个副本接收到该消息；默认 acks = 1；（Broker端只要主分区写入成功，就可以给客户端去回送响应，**如果leader宕机了，则会丢失数据** ）

  ![这里写图片描述](https://gitee.com/littlefxc/oss/raw/master/images/format,png.png)

- acks = 0：生产者发送消息之后不需要等待任何服务端的响应；（这种情况下数据传输效率最高，但是数据可靠性确是最低的。）

- acks = -1 或者 acks=all：生产者在发送消息之后，需要等待 ISR 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。

  你以为这样就能保证数据不丢失了吗？例如当ISR中的成员只有leader的时候，就相当于 acks=1 了。

  那么该怎么样保证数据的可靠性能？还需要`min.insync.replicas`这个参数(可以在broker或者topic层面进行设置)的配合，这样才能发挥最大的功效。

  - min.insync.replicas这个参数设定ISR中的最小副本数是多少，默认值为1，当且仅当request.required.acks参数设置为-1时，此参数才生效。

    如果ISR中的副本数少于`min.insync.replicas`配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。
    
  - ISR中的flower全部完成数据同步后，leader此时挂掉，会重新选举leader，数据不会丢失。

  ![这里写图片描述](https://gitee.com/littlefxc/oss/raw/master/images/format,png-20210322221324553.png)
  
  - 数据发送到leader后 ，部分ISR的副本同步，leader此时挂掉。比如follower1和follower2都有可能变成新的leader, producer端会得到返回异常，producer端会重新发送数据，数据可能会重复。
  
  ![这里写图片描述](https://gitee.com/littlefxc/oss/raw/master/images/format,png-20210322221608062.png)

- max.request.size:该参数用来限制生产者客户端能发送的消息的最大值，默认 1M（10485768）
- retries 和 retry.backoff.msretries: 重试次数和重试间隔，默认100
- compression.type: 这个参数用来指定消息的压缩方式，默认值为 none ， 可选配置：gzip，snappy 和 lz4
- connections.max.idle.ms:这个参数用来指定在多久之后关闭限制的连接，默认值是54000ms，即9分钟
- linger.ms：这个参数用来指定生产者发送 ProducerBatch 之前等待更多消息（ProducerRecord）加入ProducerBatch的时间，默认值为0
- batch.size:累计多少条消息，则一次进行批量发送
- buffer.memory:缓存提升性能参数，默认 32 M
- receive.buffer.bytes: 这个参数用来设置Socket接受消息缓冲区（SO_RECBUF）的大小，默认值为32678（B），即32KB
- send.buffer.bytes: 这个参数用来设置Socket发送消息缓存区（SO_SNDBUF）的大小，默认值为131072（B），即128KB。
- request.timeout.ms: 这个参数用来配置Producer等待请求响应的最长时间，默认值为 3000 ms

## Kafka 之拦截器

拦截器（interceptor）：Kafka对应着有生产者和消费者两种拦截器

生产者实现接口：org.apache.kafka.clients.producer.ProducerInterceptor

消费者实现接口：org.apache.kafka.clients.consumer.ConsumerInterceptor

## Kafka 之序列化和反序列化

## Kafka 之分区器

## Kafka 之消费器

## Kafka 高级应用整合 Spring Boot

# Kafka 海量日志收集系统实战

# 参考资源

[kafka数据可靠性深度解读](https://blog.csdn.net/u013256816/article/details/71091774)