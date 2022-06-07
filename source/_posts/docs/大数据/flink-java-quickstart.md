---
title: 一个 Flink 简单的入门 quickstart
date: 2021-04-06 20:14:03
categories: 大数据
tags:
- java
- flink
---

[TOC]

## Mac OS 安装 flink

```sh
brew install apache-flink
```

## 启动 flink

```sh
/usr/local/opt/apache-flink/libexec/bin/start-cluster.sh
```

查看 flink 的 Web 界面 [http://localhost:8081/](http://localhost:8081/)

![flink_quickstart_launch_flink.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/flink_quickstart_launch_flink.png)

## 新建一个 maven 工程

### 依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-java_2.12</artifactId>
    <version>1.11.0</version>
</dependency>
```

### SocketTextStreamWordCount

```java
package org.fengxuechao.example.flink.quickstart;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.util.Arrays;

/**
 * @author fengxuechao
 * @date 2021/4/6
 */
public class SocketTextStreamWordCount {
    public static void main(String[] args) throws Exception {
        //参数检查
        if (args.length != 2) {
            System.err.println("USAGE:\nSocketTextStreamWordCount <hostname> <port>");
            return;
        }

        String hostname = args[0];
        Integer port = Integer.parseInt(args[1]);


        // set up the streaming execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //获取数据
        DataStreamSource<String> stream = env.socketTextStream(hostname, port);

        //计数
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = stream.map(value -> {
            System.out.println("map:" + value);
            return value;
        }).flatMap(new LineSplitter())
                .keyBy(0)
                .sum(1);

        sum.print();

        env.execute("Java WordCount from SocketTextStream Example");
    }

    public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) {
            System.out.println("LineSplitter.value:"+s);
            String[] tokens = s.toLowerCase().split("\\W+");
            System.out.println("LineSplitter.value.splits:"+ Arrays.toString(tokens));
            for (String token : tokens) {
                if (token.length() > 0) {
                    collector.collect(new Tuple2<String, Integer>(token, 1));
                }
                System.out.println("LineSplitter.value.split:"+ token);
            }
        }
    }
}
```

## 运行

### maven 项目打包

```sh
mvn clean package
```

### 监听 9000 端口

```sh
nc -l 9000
```

![img.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/flink_quickstart_nc.png)

### 使用 flink 运行程序

```sh
flink run -c org.fengxuechao.example.flink.quickstart.SocketTextStreamWordCount flink/quickstart/target/quickstart-1.0-SNAPSHOT.jar 127.0.0.1 9000
```

### 查看 flink 界面的日志

![img.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/flink_quickstart_webui_log.png)
