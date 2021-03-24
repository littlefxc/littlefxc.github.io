---
title: elasticsearch
date: 2019-08-07 19:41:34
categories: ELK
tags: es
---

## 解决Elasticsearch-head插件链接不上服务

修改elasticsearch.yml文件

```yaml
vim $ES_HOME$/config/elasticsearch.yml
# 增加如下字段
http.cors.enabled: true
http.cors.allow-origin: "*"
```

重启ES
