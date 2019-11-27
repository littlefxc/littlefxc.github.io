---
title: ElasticSearch 安装中文分词插件
date: 2019-11-27 09:45:05
categories: elasticsearch
tags:
- 插件
- 中文分词
---

# 1. 安装 Elasticsearch

略

# 2. 安装IK分词器插件

进入 Elasticsearch 安装目录

使用 `elasticsearch-plugin` 安装插件

    ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.1/elasticsearch-analysis-ik-7.1.1.zip

安装步骤截图如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191105170349939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

**Tips**: 如果是集群，则每个 es 节点都要安装该插件

# 3. 确认安装插件成功

    ./bin/elasticsearch-plugin list
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191105170358335.png)

# 4. 移除插件

    ./bin/elasticsearch-plugin remove analysis-ik