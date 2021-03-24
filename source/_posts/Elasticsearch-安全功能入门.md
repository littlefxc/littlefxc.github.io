---
title: Elasticsearch 安全功能入门
date: 2019-11-27 09:46:37
categories: ELK
tags:
- 安全
- es
---

# 1. 前言

**[从 Elastic Stack 6.8 和 7.1 开始](https://www.elastic.co/cn/blog/security-for-elasticsearch-is-now-free)**，Elasticsearch在默认分发包中免费提供多项安全功能，例如 **TLS 加密通信**、**基于角色的访问控制 (RBAC)**，等等。在本文中，我将会演示如何启用这些功能来确保您的 Elasticsearch 集群的安全。

实际演示中，我将会在两台centos7上各自创建一个一节点 Elasticsearch 集群并进行安全设置。要实现这一点，我们首先需要在两个节点之间配置 TLS 通信。然后，我会为 Kibana 实例启用安全功能。再然后，我会在 Kibana 中配置基于角色的访问控制，从而确保用户只能看到他们获授权能够看到的内容。

尽管关于安全功能的运行过程还有很多内容，但现在我们仅会介绍入门所需知识。

# 2. 安装 Elasticsearch 和 Kibana

略

# 3. 传输层配置 TLS 和身份验证

## 3.1. 在 Elasticsearch 主节点上配置 TLS

我要做的第一件事是生成证书，通过这些证书便能允许节点安全地通信。您可以使用企业 CA 来完成这一步骤，但是在此演示中，我将会使用一个名为 elasticsearch-certutil 的命令，通过这一命令，就无需担心证书通常带来的任何困扰，便能完成这一步。

    bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""

如果您使用密码保护了节点证书的安全，请将密码添加到您的Elasticsearch密钥库中：

    bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass "testpassword"
    
    bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
    
    bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password

接下来，使用您最常用的文本编辑器打开文件 `config/elasticsearch.yaml`。将下列代码行粘贴到文件末尾。

    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

保存文件，现在我们便可以启动主节点了。运行命令 bin/elasticsearch。这一可执行文件必须保持运行，现在可以将此终端放在一边。

## 3.2. Elasticsearch 集群密码

[`elasticsearch-setup-passwords` 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/setup-passwords.html)

注意：`elasticsearch-setup-passwords` 这个命令只能使用一次。

    # 生成随机密码
    bin/elasticsearch-setup-passwords auto
    
    # 手动定义密码（建议使用）
    bin/elasticsearch-setup-passwords interactive


但是如果完全忘记了 Elasticsearch 的超级用户的密码，请看

[Elasticsearch 7.1 重置超级用户的密码](https://www.notion.so/a9cab5834874407681edc7b573730e0d)

## 3.3. 在从节点上配置 TLS

复制证书文件，然后将 **xpack.security.*** 键设置为与主节点一模一样。然后通过运行 `bin/elasticsearch` 来启动节点。我们将看到其加入集群。而且，如果看一下主节点的终端窗口，我们会看到有一条消息显示已有一个节点加入集群。现在，我们的两节点集群便开始运行了。

## 3.4. 在 Kibana 中实现安全性

在 `kibana` 安装目录中编辑 `config/kibana.yml`到类似下面的代码行

    #elasticsearch.username: "kibana"
    #elasticsearch.password: "testpassword"

对 `username` 和 `password` 字段取消注释，方法是删除代码行起始部分的 `#` 符号。将 "user" 更改为 "kibana"，然后将 "pass" 更改为 `setup-passwords` 命令告诉我们的任何 Kibana 密码。保存文件，然后我们便可通过运行 bin/kibana 启动 Kibana 了。
