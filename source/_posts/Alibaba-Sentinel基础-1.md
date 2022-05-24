---
title: Alibaba Sentinel基础(1)
date: 2021-01-21 14:41:56
categories: 微服务
tags: alibaba sentinel
---

[TOC]

# 1. 前言

Alibaba Sentinel 基础部分主要分六个部分：

- [x] Sentinel 全景分析
- [ ] Sentinel 极速入门
- [ ] Sentinel 核心API
- [ ] Sentinel 控制台使用
- [ ] Sentinel 与 Spring 整合 AOP 分析
- [ ] Sentinel 与 Dubbo 整合分析

# 2. Sentinel 全景分析

[官方介绍](https://github.com/alibaba/Sentinel/wiki/介绍)

## Sentinel 是什么？

Sentinel 是面向分布式服务架构的流量控制组件，主要以流量为切入点，从限流、流量整形、熔断降级、系统负载保护、热点防护等多个维度来帮助开发者保障微服务的稳定性。

它可以做到以下几点：

- 流量控制

- 熔断降级

- 系统保护

- 服务安全

## Sentinel 的特性

- **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- **完善的 SPI 扩展点**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/sentinel的主要特性.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/sentinel的主要特性.png)

部分是 sentinel-dashboard。

绿色部分是 sentinel-core，是 Sentinel 的核心库。

紫色部分表示 sentinel 可以持久化规则到配置中心。

蓝色部分 sentinel 的扩展点。

## Sentinel 的生态

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/sentinel_2_6.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/sentinel_2_6.png)

## Sentinel 的组成部分

- 核心库：sentinel-core
- 控制台：sentinel-dashboard
- 第三方支持组件：如spring、dubbo、rocketmq等

# 3. Sentinel 极速入门

# 4. Sentinel 核心API

# 5. Sentinel 控制台使用

# 6. Sentinel 与 Spring 整合 AOP 分析

# 7. Sentinel 与 Dubbo 整合分析