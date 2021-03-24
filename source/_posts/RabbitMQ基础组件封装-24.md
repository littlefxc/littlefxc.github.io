---
title: RabbitMQ基础组件封装(24)
date: 2021-01-27 22:02:43
categories: mq
tags: 
- mq
- rabbitmq
---

[TOC]

# 1. 整体功能概述

![rabbitmq_24_1](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_24_1.png)

![rabbitmq_24_2](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_24_2.png)

# 2. 基础组件模块划分

```java
rabbit-parent
- rabbit-api           对外提供统一的API接口
- rabbit-commmon       公共包
- rabbit-core-producer 核心包
- rabbit-task          es-job
```

[Github 地址](https://github.com/littlefxc/foodie.git)

# 3. 可靠性消息投递

![rabbitmq_19_13](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_19_13.png)

![rabbitmq_24_4](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_24_4.png)

# 4. 思维导图

![rabbitmq_xmind.png](https://gitee.com/littlefxc/oss/raw/master/images/rabbitmq_xmind.png)