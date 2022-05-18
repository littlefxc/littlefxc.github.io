---
title: Keepalived原理和配置详解
date: 2021-07-08 15:50:31
tags: 
- keepalived
---

# 1 Keepalived 简介
Keepalived 软件起初是专为 LVS 负载均衡软件设计的，用来管理并监控 LVS 集群系统中各个服务节点的状态，后来又加入了可以实现高可用的 VRRP 功能。因此，Keepalived除了能够管理 LVS 软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL等）的高可用解决方案软件。

Keepalived 软件主要通过 VRRP 协议实现高可用功能的，VRRP 是 Virtual Router Redundancy Protocol （虚拟路由器冗余协议）的缩写，VRRP 出现的目的就是为了解决动态路由单点故障问题的，它能够保证当个别节点宕机时，整个网络可以不间断的运行。所以，Keepalived 一方面具有配置管理 LVS 的功能，同时还具有对LVS 下面节点进行健康检查的功能，另一方面也可以实现系统网络服务的高可用功能。

Keepalived  软件的官方站点： http://www.keepalived.org

# 2 Keepalived 服务的三个重要功能

## 2.1 管理 LVS 负载均衡软件

早期的 LVS 软件，需要通过命令行或脚本实现管理，并且没有针对 LVS 节点的健康检查功能。为了解决 LVS 的这些使用不便的问题，Keepalived就诞生了，可以说，Keepalived软件起初是专为了解决 LVS 的问题而诞生的。因此，Keepalived和LVS的感情很深，它们的关系如同夫妻一样，可以紧密的结合，愉快的工作。Keepalived 可以通过读取自身的配置文件实现通过更底层的接口直接管理 LVS 的配置以及控制服务的启动、停止等功能，这使得 LVS 的应用就更加简单方便了。

## 2.2 实现对 LVS 集群节点健康检查功能（healthcheck）

Keepalived 可以通过在自身的keepalived.conf文件里配置 LVS 的节点 IP 和相关参数实现对 LVS 的直接管理；除此之外，当 LVS 集群中的某一个甚至是几个节点服务器同时发生故障无法提供服务时，Keepalived 服务会自动将失效的节点服务器从 LVS 的正常转发队列中清楚出去，并转换到别的正常节点服务器上，从而保证最终用户的访问不受影响；当故障的节点服务器被修复后，Keepalived 服务又会自动地把它们加入到正常转发队列中，对客户提供服务。

## 2.3作为系统网络服务的高可用功能（failover）

Keepalived 可以实现任意两台主机之间，例如 Master 和 Backup 主机之间的故障转移和自动切换，这个主机可以是普通的不能停机的业务服务器，也可以是 LVS 负载均衡、Nginx 反向代理这样的服务器。

Keepalived 高可用功能实现的原理为：两台主机同时安装好 keepalived 软件并启动服务，开始正常工作时，由角色为 Master 的主机获得所有资源并对用户提供服务，角色 Backup 的主机作为 Master 主机的热备；当角色为 Master 的主机失效或出现故障时，角色为 Backup 的主机将自动接管 Master 主机的所有工作，包括接管 VIP 资源及相应资源服务；而当角色为 Master 的主机故障修复后，又会自动接管回它原来处理的工作，角色为 Backup 的主机则同时释放 Master 主机失效它接管的工作，此时，两台主机将恢复到最初启动时各自的原始角色及工作状态。


# 3 运行原理
Keepalived 高可用服务对之间的故障切换转移，是通过 VRRP 协议（虚拟路由冗余协议）来实现的。

在 Keepalived 服务正常工作时，主 Master 节点会不断地向备节点发送（多播的方式）心跳消息，用以告诉备 Backup 节点自己还活着，当主 Master 节点发生故障时，就无法发送心跳消息了，备节点也就因此无法继续检测到来自Master 节点的心跳了，进而调用自身的接管程序，接管主 Master 节点的 IP 资源及服务。而当主 Master 节点恢复时，备 Backup 节点又会释放主节点故障时自身接管的 IP 资源及服务，恢复到原来备用角色。

# 4 选举策略
选举策略是根据 VRRP 协议，完全按照权重大小，权重最大（0～255）的是 MASTER 机器，下面几种情况会触发选举
1. keepalived 启动的时候
2. master 服务器出现故障（断网，重启，或者本机器上的 keepalived crash 等，而本机器上其他应用程序 crash 不算）
3. 有新的备份服务器加入且权重最大

# 5 VRRP协议

VRRP 协议，全称 Virtual Router Redundancy Protocol，中文名为虚拟路由冗余协议，VRRP 的出现就是为了解决静态路由的单点故障问题，VRRP 协议是通过一种竞选机制来将路由的任务交给某台 VRRP 路由器的。VRRP 协议早期是用来解决交换机、路由器等设备单点故障的。

## 5.1 VRRP 原理描述（同样适用于 Keepalived 的工作原理）

在一组 VRRP 路由器集群中，有多台物理 VRRP 路由器，但是这多台物理的机器并不是同时工作的，而是由一台称为 MASTER 的机器负责路由工作，其他的机器都是 BACKUP。MASTER 角色并非一成不变，VRRP 协议会让每个 VRRP 路由参与竞选，最终获胜的就是 MASTER。MASTER 拥有虚拟路由器的 IP 地址，我们把这个 IP 地址称为 VIP，MASTER 负责转发发送给网关地址的数据包和响应 ARP 请求。

## 5.2 VRRP 是如何工作的？

VRRP 协议通过竞选机制来实现虚拟路由器的功能，所有的协议报文都是通过 IP 多播（默认的多播地址：224.0.0.18）形式进行发送。虚拟路由器由 VRID （范围0-255）和一组 IP 地址组成，对外表现为一个周知的 MAC 地址：00-00-5E-00-01-{VRID}。所以，在一个虚拟路由器中，不管谁是 MASTER，对外都是相同的 MAC 地址和 IP 地址，如果其中一台虚拟路由器宕机，角色发生切换，那么客户端并不需要因为 MASTER 的变化修改自己的路由设置，可以做到透明的切换。这样就实现了如果一台机器宕机，那么备用的机器会拥有 MASTER 上的 IP 地址，实现高可用功能。

## 5.3 VRRP 是如何通信的？

在一组虚拟路由器中，只有作为 MASTER 的 VRRP 路由器会一直发送 VRRP 广播包，此时 BACKUP 不会抢占 MASTER 。当 MASTER 不可用时，这个时候 BACKUP  就收不到来自 MASTER 的广播包了，此时多台 BACKUP 中优先级最高的路由器会去抢占为 MASTER。这种抢占是非常快速的（可能只有1秒甚至更少），以保证服务的连续性。出于安全性考虑，VRRP 数据包使用了加密协议进行了加密。

# 6 Keepalived 高可用服务脑裂问题

## 6.1 什么是脑裂？

由于某些原因，导致两台高可用服务器在指定时间内，无法检测到对方的心跳消息，各自取得资源及服务的所有权，而此时的两台高可用服务器都还活着并在正常运行，这样就会导致同一个 IP 或服务在两端同时存在发生冲突，最严重的是两台主机占用同一个 VIP 地址，当用户写入数据时可能会分别写入到两端，这可能会导致服务器两端的数据不一致或造成数据丢失，这种情况就被称为脑裂。

## 6.2 导致脑裂发生的原因

一般来说，脑裂的发生，有以下几种原因：

1）高可用服务器之间心跳线链路故障，导致无法正常通信。

心跳线坏了（包括断了，老化）
网卡及相关驱动坏了，IP 配置及冲突问题（网卡直连）
心跳线连接的设备故障（网卡及交换机）
2）高可用服务器上开启了 iptables 防火墙阻挡了心跳消息传输。

3）高可用服务器上心跳网卡地址等信息配置不正确，导致发送心跳失败。

4）其他服务配置不当等原因，如心跳方式不同，心跳广播冲突、软件 BUG等。

注意：Keepalived 配置里同一 VRRP 实例如果 virtual_router_id 参数两端配置不一致，也会导致脑裂问题发生。

## 6.3 解决脑裂的具体方案

在实际生产环境中，可以从以下几个方面来防止脑裂问题的发生

1）同时使用串行电缆和以太网电缆连接，同时用两条心跳线路，这样一条线路坏了，另一个还是好的，依然能够传送心跳消息

2）当检测到脑裂时强行关闭一个心跳节点（这个功能需要特殊设备支持，如Stonith、fence）。相当于备节点接收不到心跳消息，发送关机命令通过单独的线路关闭主节点的电源。

3）做好对脑裂的监控报警（如邮件及手机短信等或值班），在问题发生时人为第一时间介入仲裁，降低损失。例如，百度的监控报警短信就有上行和下行的区别。报警信息报到管理员手机上，管理员可以通过手机回复对应数字或简单的字符串操作返回给服务器，让服务器根据指令自动处理相应故障，这样解决故障的时间更短。

4）如果开启防火墙，一定要让心跳消息通过，一般通过允许 IP 段的形式。

# 7 KeepAlived 配置详解

Keepalived的所有配置都在一个配置文件里面，主要分为三类：

- 全局配置
- VRRPD配置
- LVS 配置

## 7.1 全局配置

全局配置是对整个 Keepalived 生效的配置，一个典型的配置如下：

```
global_defs {
   notification_email {         #设置 keepalived 在发生事件（比如切换）的时候，需要发送到的email地址，可以设置多个，每行一个。
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc    #设置通知邮件发送来自于哪里，如果本地开启了sendmail的话，可以使用上面的默认值。
   smtp_server 192.168.200.1    #指定发送邮件的smtp服务器。
   smtp_connect_timeout 30      #设置smtp连接超时时间，单位为秒。
   router_id LVS_DEVEL          #是运行keepalived的一个表示，多个集群设置不同。
}
```

## 7.2 VRRPD配置

VRRPD 的配置是 Keepalived 比较重要的配置，主要分为两个部分 VRRP 同步组和 VRRP实例，也就是想要使用 VRRP 进行高可用选举，那么就一定需要配置一个VRRP实例，在实例中来定义 VIP、服务器角色等。

### 7.2.1 VRRP Sync Groups**

不使用Sync Group的话，如果机器（或者说router）有两个网段，一个内网一个外网，每个网段开启一个VRRP实例，假设VRRP配置为检查内网，那么当外网出现问题时，VRRPD认为自己仍然健康，那么不会发生Master和Backup的切换，从而导致了问题。Sync group就是为了解决这个问题，可以把两个实例都放进一个Sync Group，这样的话，group里面任何一个实例出现问题都会发生切换。

```
vrrp_sync_group VG_1{ #监控多个网段的实例
group {
　　　　VI_1 #实例名
　　　　VI_2
　　　　......
}
notify_master /path/xx.sh 　　　　#指定当切换到master时，执行的脚本
netify_backup /path/xx.sh 　　　　#指定当切换到backup时，执行的脚本
notify_fault "path/xx.sh VG_1"   #故障时执行的脚本
notify /path/xx.sh
smtp_alert 　　#使用global_defs中提供的邮件地址和smtp服务器发送邮件通知
}
```

### 7.2.2 VRRP实例（instance）配置

VRRP实例就表示在上面开启了VRRP协议，这个实例说明了VRRP的一些特征，比如主从，VRID等，可以在每个interface上开启一个实例。

```
vrrp_instance VI_1 {
    state MASTER         #指定实例初始状态，实际的MASTER和BACKUP是选举决定的。
    interface eth0       #指定实例绑定的网卡
    virtual_router_id 51 #设置VRID标记，多个集群不能重复(0..255)
    priority 100         #设置优先级，优先级高的会被竞选为Master，Master要高于BACKUP至少50
    advert_int 1         #检查的时间间隔，默认1s
    nopreempt            #设置为不抢占，说明：这个配置只能在BACKUP主机上面设置
    preempt_delay        #抢占延迟，默认5分钟
    debug                #debug级别
    authentication {     #设置认证
        auth_type PASS    #认证方式，支持PASS和AH，官方建议使用PASS
        auth_pass 1111    #认证的密码
    }
    virtual_ipaddress {     #设置VIP，可以设置多个，用于切换时的地址绑定。格式：#<IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPT> label <LABE
        192.168.200.16/24 dev eth0 label eth0:1
        192.168.200.17/24 dev eth1 label eth1:1
        192.168.200.18
    }
}
```

### 7.2.3 VRRP 脚本 

```
# VRRP 脚本 
# 如下所示为相关配置示例
vrrp_script check_running {
   script "/usr/local/bin/check_running"
   interval 10
   weight 10
}

vrrp_instance http {
   state BACKUP
   smtp_alert
   interface eth0
   virtual_router_id 101
   priority 90
   advert_int 3
   authentication {
   auth_type PASS
   auth_pass whatever
   }
   virtual_ipaddress {
   1.1.1.1
   }
   track_script {
   check_running 
   }
}
# 首先在 vrrp_script 区域定义脚本名字和脚本执行的间隔和脚本执行的优先级变更, 如下所示:
vrrp_script check_running {
            script "/usr/local/bin/check_running"
            interval 10     # 脚本执行间隔
            weight 10       # 脚本结果导致的优先级变更 10 表示优先级 + 10-10 则表示优先级 - 10
            }
# 然后在实例(vrrp_instance) 里面引用有点类似脚本里面的函数引用一样先定义后引用函数名
track_script {
      check_running 
}
```

注意:
VRRP 脚本 (vrrp_script) 和 VRRP 实例 (vrrp_instance) 属于同一个级别
keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级。一般脚本检测返回的值为 0，说明脚本检测成功，如果为非 0 数值，则说明检测失败
如果脚本执行结果为 0，并且 weight 配置的值大于 0，则优先级相应的增加, 如果 weight 为非 0，则优先级不变
如果脚本执行结果非 0，并且 weight 配置的值小于 0，则优先级相应的减少, 如果 weight 为 0，则优先级不变
其他情况，维持原本配置的优先级，即配置文件中 priority 对应的值。
这里需要注意的是：
1） 优先级不会不断的提高或者降低
2） 可以编写多个检测脚本并为每个检测脚本设置不同的 weight
3） 不管提高优先级还是降低优先级，最终优先级的范围是在[1,254]，不会出现优先级小于等于 0 或者优先级大于等于 255 的情况
这样可以做到利用脚本检测业务进程的状态，并动态调整优先级从而实现主备切换。

## 7.3 LVS 配置

虚拟服务器virtual_server定义块 ，虚拟服务器定义是keepalived框架最重要的项目了，是keepalived.conf必不可少的部分。 该部分是用来管理LVS的，是实现keepalive和LVS相结合的模块。ipvsadm命令可以实现的管理在这里都可以通过参数配置实现，注意：real_server是被包含在viyual_server模块中的，是子模块。

```
virtual_server 192.168.202.200 23 {        //VIP地址，要和vrrp_instance模块中的virtual_ipaddress地址一致
　　　　delay_loop 6   #健康检查时间间隔
　　　　lb_algo rr 　　#lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
　　　　lb_kind DR    #负载均衡转发规则NAT|DR|RUN
　　　　persistence_timeout 5 #会话保持时间
　　　　protocol TCP    #使用的协议
　　　　persistence_granularity <NETMASK> #lvs会话保持粒度
　　　　virtualhost <string>    #检查的web服务器的虚拟主机（host：头）
　　　　sorry_server<IPADDR> <port> #备用机，所有realserver失效后启用

real_server 192.168.200.5 23 {             //RS的真实IP地址
            weight 1 #默认为1,0为失效
            inhibit_on_failure #在服务器健康检查失效时，将其设为0，而不是直接从ipvs中删除
            notify_up <string> | <quoted-string> #在检测到server up后执行脚本
            notify_down <string> | <quoted-string> #在检测到server down后执行脚本

TCP_CHECK {                    //常用
            connect_timeout 3 #连接超时时间
            nb_get_retry 3 #重连次数
            delay_before_retry 3 #重连间隔时间
            connect_port 23  #健康检查的端口的端口
            bindto <ip>
          }

HTTP_GET | SSL_GET{          //不常用
    url{ #检查url，可以指定多个
         path /
         digest <string> #检查后的摘要信息
         status_code 200 #检查的返回状态码
        }
    connect_port <port>
    bindto <IPADD>
    connect_timeout 5
    nb_get_retry 3
    delay_before_retry 2
    }

SMTP_CHECK{                 //不常用
    host{
    connect_ip <IP ADDRESS>
    connect_port <port> #默认检查25端口
    bindto <IP ADDRESS>
         }
    connect_timeout 5
    retry 3
    delay_before_retry 2
    helo_name <string> | <quoted-string> #smtp helo请求命令参数，可选
    }

MISC_CHECK{                 //不常用
    misc_path <string> | <quoted-string> #外部脚本路径
    misc_timeout #脚本执行超时时间
    misc_dynamic #如设置该项，则退出状态码会用来动态调整服务器的权重，返回0 正常，不修改；返回1，

　　检查失败，权重改为0；返回2-255，正常，权重设置为：返回状态码-2
    }
}
```
