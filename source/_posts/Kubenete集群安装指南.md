---
title: Kubernetes集群安装指南
date: 2020-04-27 15:00:31
tags: [k8s, 安装指南]
categories: k8s
---

## 准备工作

### 查看系统版本

```sh
[root@k8s-master-1 ~]# cat /etc/centos-release
CentOS Linux release 7.6.1810 (Core)
```

**overlay2介绍**

overlay的改进版，只支持4.0以上内核添加了Multiple lower layers in overlayfs的特性，所以overlay2可以直接造成muitiple lower layers不用像overlay一样要通过硬链接的方式(最大128层) centos的话支持3.10.0-514及以上内核版本也有此特性，所以消耗更少的inode

docker官方overlay2的PR:
[https://github.com/moby/moby/pull/22126](https://github.com/moby/moby/pull/22126)

LINUX KERNERL 4.0 release说明：
[https://kernelnewbies.org/Linux_4.0](https://kernelnewbies.org/Linux_4.0)

### 配置主机名

为将来要作为主节点的服务器设置主机名。

```sh
hostnamectl set-hostname k8s-master-1 --static
```

### 配置服务器hosts

各个服务器上都要配置

```sh
[root@k8s-master-1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 k8s-master-1
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.212.155 salt

0.0.0.0 aliyun.one
0.0.0.0 lsd.systemten.org
0.0.0.0 pastebin.com
0.0.0.0 pm.cpuminerpool.com
0.0.0.0 systemten.org

192.168.200.19 k8s-master-1
192.168.154.14 ip-192-168-154-14
192.168.154.15 ip-192-168-154-15
192.168.154.16 ip-192-168-154-16
```

### 关闭swap，注释swap分区

```sh
[root@k8s-master-1 ~]# swapoff -a
[root@k8s-master-1 ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Mon Mar 26 20:43:49 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=848d5a8b-0ee9-481f-b1ff-833fb35cfd03 /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

### 添加网易 yum 镜像

```sh
[root@k8s-master-1 ~]# cat /etc/yum.repos.d/CentOS-Base.repo
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#
[base]
name=CentOS-$releasever - Base - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
baseurl=http://mirrors.163.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://mirrors.163.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://mirrors.163.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - 163.com
baseurl=http://mirrors.163.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
```

### 关闭防火墙

在各个服务器上关闭防火墙

```sh
[root@k8s-master-1 ~]# systemctl stop firewalld.service            #停止firewall
[root@k8s-master-1 ~]# systemctl disable firewalld.service         #禁止firewall开机启动
[root@k8s-master-1 ~]# systemctl stop iptables.service             #停止iptables
[root@k8s-master-1 ~]# systemctl disable iptables.service          #禁止iptables开机启动l
```

### 配置内核参数，将桥接的IPv4流量传递到iptables的链

各个服务器都要配置

```sh
[root@k8s-master-1 ~]# cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 禁用SELinux

```sh
[root@k8s-master-1 ~]# sestatus
SELinux status: disabled
```

## 安装配置 docker

官方文档地址：[Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

Docker官方文档对安装步骤描述已经足够详细, 过程并不复杂, 本文便不再赘述.

### 安装 docker

本文安装docker的版本是`18.09`, 安装时请按照文档描述的方式明确指定版本号`yum install docker-ce-18.09.9-3.el7 docker-ce-cli-18.09.9-3.el7 containerd.io`.

### 配置 docker

官方文档地址：[容器运行时](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/)

```sh
[root@k8s-master-1 ~]# cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://registry.docker-cn.com", "https://docker.mirrors.ustc.edu.cn", "https://fzhifedh.mirror.aliyuncs.com"],
  "insecure-registries": ["hub.51iwifi.com","alpha-harbor.51iwifi.com","192.168.195.2","134.108.20.13"],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
    }
}
```

### 安装完后重启

```sh
[root@k8s-master-1 ~]# systemctl enable docker
[root@k8s-master-1 ~]# systemctl start docker
```

同样在各个服务器上都要保持一致

## 安装 Kubernetes(kubectl, kubelet, kubeadm)

### 添加阿里kubernetes源

```sh
[root@k8s-master-1 ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

### 安装

```sh
[root@k8s-master-1 ~]# yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
[root@k8s-master-1 ~]# systemctl enable kubelet
```

### 初始化 master 节点

```sh
[root@k8s-master-1 ~]# kubeadm config print init-defaults > kubeadm-init.yaml
```

该文件有两处需要修改:

将`advertiseAddress: 1.2.3.4`修改为本机地址
将`imageRepository: k8s.gcr.io`修改为`imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers`

修改完毕后文件如下:

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 0.0.0.0
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

### 下载镜像

```sh
[root@k8s-master-1 ~]# kubeadm config images pull --config kubeadm-init.yaml
```

### 执行初始化

```sh
[root@k8s-master-1 ~]# kubeadm config images pull --config kubeadm-init.yaml
```

等待执行完毕后, 会输出如下内容:

```sh
...
Your Kubernetes control-plane has initialized successfully!
...
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.200.19:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:a67698bdd29af4af0d70a563c4a17d1c751faabe65d7d3661eb90783568ecda6
```

最后两行需要保存下来, `kubeadm join ...`是其它worker节点加入所需要执行的命令.

接下来配置环境, 让当前用户可以执行kubectl命令:

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看节点，`kubectl get node`

```sh
[root@k8s-master-1 ~]# kubectl get node
NAME           STATUS     ROLES    AGE     VERSION
k8s-master-1   NotReady   master   3m25s   v1.18.0
```

node节点为`NotReady`，因为 pod `coredns` 没有启动，缺少网络pod.

## 安装 calico 网络

官方文档地址：[Instructions](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

### 下载 calico 的 k8s 文件

```sh
[root@k8s-master-1 ~]# wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
[root@k8s-master-1 ~]# cat kubeadm-init.yaml | grep serviceSubnet:
  serviceSubnet: 10.96.0.0/12
```

打开 `calico.yaml`, 将`192.168.0.0/16`修改为`10.96.0.0/12`

> 需要注意的是, calico.yaml中的IP和kubeadm-init.yaml需要保持一致, 要么初始化前修改kubeadm-init.yaml, 要么初始化后修改calico.yaml.

执行`kubectl apply -f calico.yaml`初始化网络.

此时查看node信息, master的状态已经是`Ready`了.

```sh
[root@k8s-master-1 ~]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master-1   Ready    master   15m   v1.18.0
```

## 安装 dashboard

### 部署 dashboard

官方文档：[网页界面 (Dashboard)](https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/)

官方部署dashboard的服务没使用nodeport，将yaml文件下载到本地，在service里添加`NodePort`

### 创建用户

官方文档地址: [Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

创建一个用于登录Dashboard的用户. 创建文件`dashboard-adminuser.yaml`内容如下:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

执行命令`kubectl apply -f dashboard-adminuser.yaml`.

### 登录

官方文档地址：[Bearer Token](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

使用token进行登录，执行下面命令获取token

```sh
kubectl describe secrets -n kubernetes-dashboard kubernetes-dashboard-token-t4hxz  | grep token | awk 'NR==3{print $2}'
```

复制该Token到登录页, 点击登录即可, 效果如下:

![image](Kubenete集群安装指南/)%

## 添加其它 Worker 节点

在使用 `kubeadm` 初始化 master 节点后会有 `kubeadm join ...` 这样的返回信息，详见前文。

同时，默认你已经在其它的服务器中已经安装了 docker, kubernetes. 

请注意在其它的服务器只需安装kubernetes，等初始化 master 节点后，执行如下命令将 Worker 加入集群:

```sh
kubeadm join 192.168.200.19:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:a67698bdd29af4af0d70a563c4a17d1c751faabe65d7d3661eb90783568ecda6
```

添加完毕后, 在Master上查看节点状态:

```sh
[root@k8s-master-1 k8s-master]# kubectl get node
NAME                STATUS   ROLES    AGE   VERSION
ip-192-168-154-14   Ready    <none>   14d   v1.18.0
ip-192-168-154-15   Ready    <none>   14d   v1.18.0
ip-192-168-154-16   Ready    <none>   14d   v1.18.0
k8s-master-1        Ready    master   19d   v1.18.0
```

## 参考资源

[使用kubeadm在Centos8上部署kubernetes1.18](https://www.kubernetes.org.cn/7189.html)

[Kubernetes(一) 跟着官方文档从零搭建K8S](https://blog.piaoruiqing.com/2019/09/17/kubernetes-1-installation/)