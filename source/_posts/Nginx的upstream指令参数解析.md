---
title: Nginx的upstream指令参数解析
date: 2020-09-09 18:09:02
categories: nginx
tags: 
- upstream
- nginx
---

# **upstream 指令参数 max_conns**

限制每台server的连接数，用于保护避免过载，可起到限流作用。测试参考配置如下：

```
# worker进程设置1个，便于测试观察成功的连接数
worker_processes 1;
upstream tomcats { 
    server 192.168.1.173:8080 max_conns=2; 
    server 192.168.1.174:8080 max_conns=2; 
    server 192.168.1.175:8080 max_conns=2;
}
```

# **upstream 指令参数 slow_start**

配置了这个参数，他会覆盖权重，慢慢从0开始到正常值。

***商业版，需要付费\***

配置参考如下：

```
upstream tomcats { 
    server 192.168.1.173:8080 weight=6 slow_start=60s;
    # server 192.168.1.190:8080; 
    server 192.168.1.174:8080 weight=2; 
    server 192.168.1.175:8080 weight=2;
}
```

注意：

- 该参数不能使用在`hash` 和 `random load balancing` 中。
- 如果在 upstream 中只有一台 server，则该参数失效。

# **upstream 指令参数 down、backup**

**`down`** 用于标记服务节点不可用：

```
upstream tomcats { 
	server 192.168.1.173:8080 down;
	# server 192.168.1.190:8080; 
	server 192.168.1.174:8080 weight=1; 
	server 192.168.1.175:8080 weight=1;
}
```

------

**`backup`**表示当前服务器节点是备用机，只有在其他的服务器都宕机以后，自己才会加入到集群中，被用户访问到：

```
upstream tomcats { 
		server 192.168.1.173:8080 backup;
		# server 192.168.1.190:8080; 
		server 192.168.1.174:8080 weight=1; 
		server 192.168.1.175:8080 weight=1;
}
```

注意：

- `backup`参数不能使用在`hash` 和 `random load balancing` 中。

# **upstream 指令参数 max_fails、fail_timeout**

**`max_fails`**：表示失败几次，则标记server已宕机，剔出上游服务。

**`fail_timeout`**：表示失败的重试时间。假设目前设置如下：

```
max_fails=2 fail_timeout=15s
```

则代表在15秒内请求某一server失败达到2次后，则认为该server已经挂了或者宕机了，随后再过15秒，这15秒内不会有新的请求到达刚刚挂掉的节点上，而是会请求到正常运作的server，15秒后会再有新请求尝试连接挂掉的server，如果还是失败，重复上一过程，直到恢复。

