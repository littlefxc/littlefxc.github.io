---
title: SpringBoot整合ES的JavaRestClient
date: 2019-07-24 10:45:35
categories: elasticsearch
tags:
- spring-boot
- elasticsearch
---

## 1. 前言

首先明确一点，SpringBoot自带的ES模板，不建议使用，建议使用Rest Client。如果业务简单，且无特殊要求，可以使用SpringBoot的模板ElasticsearchRepository来搞定。这个非常简单，这里不作介绍

> ElasticsearchRepository
> - 优点： 简单，SpringBoot无缝对接，配置简单
> - 缺点： 基于即将废弃的TransportClient， 不能支持复杂的业务

## 2. 使用 Spring 的 IOC 管理 ES 的连接客户端

步骤：

1. 配置ES节点
2. 配置Rest Client
3. 配置Rest High Level Client
4. 使用IOC注入

根据我从其他网站上查询的资料，Rest Client是长连接，而且内部有默认的线程池管理，因此一般无需自定义线程池管理连接。如果不对请指正。

基于以上结论。模仿 `spring-boot-autoconfigure` 先把连接点全部配置到配置文件中.

### 2.1. 配置 maven 依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <!-- 省略 -->
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <elasticsearch.version>7.1.1</elasticsearch.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>${elasticsearch.version}</version>
        </dependency>
    </dependencies>

</project>
```

### 2.2. 编写配置类

```java
package com.awifi.capacity.admin.statistic.elasticsearch;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.List;

/**
 * Configuration properties for Elasticsearch.
 *
 * @author fengxuechao
 * @since 1.0.2
 */
@Data
@ConfigurationProperties(prefix = "elasticsearch")
public class ElasticsearchProperties {

    private List<String> hostAndPortList;

    private String username;

    private String password;
}
```

```java
package com.awifi.capacity.admin.statistic.elasticsearch;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.nio.client.HttpAsyncClientBuilder;
import org.elasticsearch.client.Client;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.ReflectionUtils;
import org.springframework.util.StringUtils;

import java.io.Closeable;
import java.util.ArrayList;
import java.util.List;

/**
 * ES 配置
 *
 * @author fengxuechao
 * @since 1.1.0
 */
@Configuration
@ConditionalOnClass({Client.class, RestHighLevelClient.class})
@EnableConfigurationProperties(ElasticsearchProperties.class)
public class ElasticsearchConfiguration implements DisposableBean {

    private static final Log logger = LogFactory.getLog(ElasticsearchConfiguration.class);

    private Closeable closeable;

    @Autowired
    private ElasticsearchProperties properties;


    /**
     * 创建 Elasticsearch RestHighLevelClient
     *
     * @return
     */
    @Bean("restHighLevelClient")
    @ConditionalOnMissingBean
    public RestHighLevelClient restHighLevelClient() {
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        List<HttpHost> list = createHttpHost();
        HttpHost[] array = list.toArray(new HttpHost[list.size()]);
        RestClientBuilder builder = RestClient.builder(array);
        //es账号密码设置
        if (StringUtils.hasText(properties.getUsername())) {
            String username = properties.getUsername();
            String password = properties.getPassword();
            UsernamePasswordCredentials usernamePasswordCredentials = new UsernamePasswordCredentials(username, password);
            credentialsProvider.setCredentials(AuthScope.ANY, usernamePasswordCredentials);
            builder.setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {

                /**
                 * 这里可以设置一些参数，比如cookie存储、代理等等
                 * @param httpClientBuilder
                 * @return
                 */
                @Override
                public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                    httpClientBuilder.disableAuthCaching();
                    return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                }
            });
        }
        // RestHighLevelClient实例需要Rest low-level client builder构建
        RestHighLevelClient restHighLevelClient = new RestHighLevelClient(builder);
        closeable = restHighLevelClient;
        return restHighLevelClient;
    }

    /**
     * 读取配置文件es信息构建 HttpHost 列表
     *
     * @return
     */
    private List<HttpHost> createHttpHost() {
        List<String> hostAndPortList = properties.getHostAndPortList();
        if (hostAndPortList.isEmpty()) {
            throw new IllegalArgumentException("必须配置elasticsearch节点信息");
        }
        List<HttpHost> list = new ArrayList<>(hostAndPortList.size());
        for (String s : hostAndPortList) {
            String[] hostAndPortArray = s.split(":");
            String hostname = hostAndPortArray[0];
            int port = Integer.parseInt(hostAndPortArray[1]);
            list.add(new HttpHost(hostname, port));
        }
        return list;
    }


    /**
     * 当不再需要时，需要关闭高级客户端实例，以便它所使用的所有资源以及底层的http客户端实例及其线程得到正确释放。
     * 通过close方法来完成，该方法将关闭内部的RestClient实例
     */
    @Override
    public void destroy() throws Exception {
        if (this.closeable != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Closing Elasticsearch client");
                }
                try {
                    this.closeable.close();
                } catch (NoSuchMethodError ex) {
                    // Earlier versions of Elasticsearch had a different method name
                    ReflectionUtils.invokeMethod(
                            ReflectionUtils.findMethod(Closeable.class, "close"),
                            this.closeable);
                }
            } catch (final Exception ex) {
                if (logger.isErrorEnabled()) {
                    logger.error("Error closing Elasticsearch client: ", ex);
                }
            }
        }
    }
}
```

### 2.3. 在 application.properties 文件中配置 es 集群信息

application.properties

```yaml
# ES 配置
elasticsearch.hostAndPortList[0]=192.168.200.19:9200
elasticsearch.username=
elasticsearch.password=
```

### 2.4. 单元测试
