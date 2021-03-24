---
title: Consul - Watches监控主动通知
date: 2021-01-13 10:04:09
categories: 注册中心
tags: consul
---

# 1 概念

当Consul中注册的服务信息发生变化的时候，我们除了定时通过接口去查询最新的服务信息之外，consul还提供了watch机制，通过监控consul数据的变化，主动通知。

目前conusl watch支持两种通知方式：**可执行程序**和**Http接口**。

consul watch支持监控的数据类型：

- services - 监控指定列表服务的可用性
- service - 监控指定服务的实例
- nodes - 监控节点
- key - 监控特定的键值对
- keyprefix - 监控consul配置中心的前缀
- checks - 监控健康检查的值
- event - 监控自定义用户事件

也就是说，consul支持监控服务、键值数据、节点信息，只要发生变化，就通知我们。

# 2 基本用法

我们以监控键值数据的变化为例子，介绍watch机制的用法。

watch监控的配置信息是跟consul的配置文件放在一起的，通过配置文件中的watches字段，设置监控信息。

说明：watches的配置，大致用法就是通过type，配置监控数据的类型，然后根据不同数据类型配置不同的参数，最后选择一种通知方式进行配置。

## 2.1 通过可执行程序通知

当我们监控的数据发生变化，consul agent会调用我们配置的可执行程序（命令、脚本等等），并且通过标准输入，以Json格式传入通知的参数， 我们只要在程序中根据参数处理业务即可。

例子：

下面是我们consul agent的配置，文件名是 `/etc/consul/watches.json` 。

```json
{
	"datacenter": "dc1", 
	"data_dir": "/var/consul", 
	"ui": true,
	"watches": [{
		"type": "key",
		"key": "foo/bar/baz",
		"handler_type": "script",
		"args": ["/usr/bin/my-service-handler.sh", "-redis"]
	}]
}
```

这个例子的意思，监控`key=foo/bar/baz`的键值数据，如果数据发生变化，就调用`/usr/bin/my-service-handler.sh -redis` 命令。

参数说明：

- datacenter - 数据中心的名字

- data_dir - consul数据存放目录

- ui - 开启consul ui

- watches - 配置监控信息，如果监控的数据发生变化，则根据配置执行通知。

  watches参数说明：

  - type - 监控的数据类型
  - key - 监控的键值数据的Key
  - handler_type - 通知类型，支持script和http
  - args - 配置通知类型为script的，执行命令，是一个数组，第一个元素是命令，后面第2个到第N个元素是命令的参数。

例子的`watches`配置的意思就是：

监控`key=foo/bar/baz`的键值数据，如果数据发生变化，就调用`/usr/bin/my-service-handler.sh -redis` 命令， 这个命令可以通过标准输入，接收变化的数据。

启动consul，就会加载watches配置。

## 2.2 通过Http接口通知

通过http接口通知数据变化，大体上配置跟上面一样，区别是多了一些http接口的配置参数。

例子：

```json
{
	"datacenter": "dc1",
	"data_dir": "/Users/jogin/Documents/work/local/consul/data",
	"ui": true,
	"watches": [{
		"type": "key",
		"key": "foo/bar/baz",
		"handler_type": "http",
		"http_handler_config": {
			"path": "<https://localhost:8000/watch>",
			"method": "POST",
			"header": {
				"x-foo": ["bar", "baz"]
			},
			"timeout": "10s",
			"tls_skip_verify": false
		}
	}]
}
```

这个例子的意思，就是当`key=foo/bar/baz`的键值数据发生变化，就通过`https://localhost:8000/watch`通知我们。

我们关注`watches`字段的配置信息，下面是参数说明：

- type - watch监控类型是key

- key - 监控foo/bar/baz这个key

- handler_type - 通知类型, http

- http_handler_config - 配置http通知信息。

  http_handler_config参数说明：

  - path - 通知Url
  - method - http请求方法
  - header - 自定义Http请求头，没有可以忽略
  - timeout - 超时时间，10秒
  - tls_skip_verify - 是否跳过tls验证

# 3 监控服务

上面介绍了watch的基本用法，我们也可以监控服务信息的变化，例如当有人注册新的服务或者服务不可用的时候，通知我们。

我们忽略掉，consul agent的配置，单独看watches的配置。

监控所有的服务的配置

```
{
	"watches": [{
		"type": "services",
		"handler_type": "http",
		"http_handler_config": {
			...忽略...
		}
	}]
}
```

监控单个服务的配置

```
{
	"watches": [{
		"type": "service",
                "service": "要监控的服务名",
		"handler_type": "http",
		"http_handler_config": {
			...忽略...
		}
	}]
}
```

# 4 监控键值数据

前面介绍过基本Key的监控，其实我们还可以通过key的前缀，批量监控一批key，只要key的前缀相同，这些Key下面的数据发生变化，都会发送通知。

```
{
	"watches": [{
		"type": "keyprefix",
		"prefix": "foo/",
		"args": ["/usr/bin/my-service-handler.sh", "-redis"]
	}]
}
```

监控类型为：keyprefix

通过prefix字段，配置key的前缀。

这个配置的意思就是：以foo/开头的Key, 数据发生变化，都会执行args参数，配置的命令。

# 5 监控节点

我们也可以监控consul集群节点的变化信息。

```
{
	"watches": [{
		"type": "nodes",
		"handler_type": "http",
		"http_handler_config": {
			...忽略...
		}
	}]
}
```

没有其他额外的参数，type=nodes即可，当节点信息发生变化，会根据配置的方式通知。

# 参考资源

[Consul by HashiCorp](https://www.consul.io/docs/dynamic-app-config/watches)

[ExplorerMan](https://www.cnblogs.com/ExMan/p/11907491.html)

[Consul Watches监控服务变化](https://www.tizi365.com/archives/535.html)

[开发者头条](https://toutiao.io/posts/zeu435/preview)

[Consul 单机集群搭建](https://www.notion.so/Consul-6982c01eb1594549ab0a6dd463f16cfd)