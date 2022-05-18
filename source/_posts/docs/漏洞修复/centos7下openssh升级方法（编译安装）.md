---
title: "centos7下openssh升级方法（编译安装）"
date: 2021-12-31T15:57:21+08:00
categories: FAQ
tags:
- openssh
---


注意：
首先打开两个或以上的shell连接，因为在升级过程中如果升级失败会导致不发新建shell连接；升级后使用xshell6,7连接，openssh版本对应修改，下载地址：https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/

1. 解压：tar zxvf openssh-8.2p1.tar.gz

2. 查看当前安装的ssh rpm -qa|grep ssh

3. 卸载当前的

    ssh rpm -e --nodeps openssh-askpass-5.3p1-94.el6.x86_64 openssh-clients-5.3p1-94.el6.x86_64 openssh-5.3p1-94.el6.x86_64 openssh-server-5.3p1-94.el6.x86_64

4. 安装依赖包 yum install -y openssl-devel

5. 删除/etc/ssh/下的密钥对，编译新的ssh
   
    rm -f /etc/ssh/ssh_host_*
    cd openssh-8.2p1
    ./configure --prefix=/usr --sysconfdir=/etc/ssh
    make && make install

6. 安装成功之后，查看版本 ssh-V

7. 修改配置文件 sshd_config 把permitrootlogin xxx去掉注释 改成 permitrootlogin yes
    把 #UseDNS no 修改 UseDNS no

8. 把sshd拷贝到 /etc/init.d下面

    cp -a contrib/redhat/sshd.init /etc/init.d/sshd
    cp -a contrib/redhat/sshd.pam /etc/pam.d/sshd.pam
    chmod +x /etc/init.d/sshd
    chkconfig --add sshd
    systemctl enable sshd
    chkconfig sshd on
    mv  /usr/lib/systemd/system/sshd.service  /opt/ （执行后如提示没有文件，忽略即可）
    mv  /usr/lib/systemd/system/sshd.socket  /opt/ （执行后如提示没有文件，忽略即可）


9.启动 /etc/init.d/sshd 后面无需加start等参数。

 