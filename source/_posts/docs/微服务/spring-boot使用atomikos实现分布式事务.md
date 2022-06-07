---
title: spring-boot使用atomikos实现分布式事务
date: 2019-05-06 19:42:16
categories: Java
tags:
- spring-boot
- atomikos
- 分布式事务
---

Java规范对分布式事务定义了标准的规范Java事务API和Java事务服务，分别是JTA和JTS一个分布式事务必须包括一个事务管理器和多个资源管理器。

资源管理器是任意类型的持久化数据存储；
而事务管理器则是承担着所有事务参与单元者的相互通讯的责任。

JTA的规范制定了分布式事务的实现的整套流程框架，定义了各个接口且只有接口，而实现分别交给事务管理器的实现方和资源管理器的实现方

<!-- more -->

## 1.前言

Java规范对分布式事务定义了标准的规范Java事务API和Java事务服务，分别是JTA和JTS一个分布式事务必须包括一个事务管理器和多个资源管理器。

资源管理器是任意类型的持久化数据存储；
而事务管理器则是承担着所有事务参与单元者的相互通讯的责任。

JTA的规范制定了分布式事务的实现的整套流程框架，定义了各个接口且只有接口，而实现分别交给事务管理器的实现方和资源管理器的实现方

对于资源管理器而言，主要包括数据库连接，JMS等，还有很多了解的不清楚。

对于事务管理器而言，从网上了解主要是应用服务器，包括JBOSS，WEBLOGIC等应用服务器，也就是说事务管理器的实现方是应用服务器，用来管理事务的通讯和协调。
对于大多数谈的数据库了解，事务管理器需要从数据库获得XAConnection , XAResource等对象，而这些对象是数据库驱动程序需要提供的，所以如果要实现分布式事务还必须有支持分布式事务的数据库服务器以及数据库驱动程序。

对Mysql而言，在mysql5.0以上的版本已经支持了分布式事务，另外常用的mysql-connector-java-5.1.25-bin.jar也是支持分布式事务的，可以在jar包的com.mysql.jdbc.jdbc2.optional中找到XA对象的实现
上面介绍了事务管理器和资源管理器的实现方式，在学习研究过程中发现对于事务管理器，特别强调了tomcat等服务器是不支持的，这句话的意思应该是在tomcat容器内
并没有分布式事务管理器的实现对象。而在JBOSS或者WEBLOGIC等商业服务器应该内置了分布式事务管理器的实现对象，应用程序可以通过JNDI方式获取UserTransaction
和TransactionManager等分布式事务环境中所需要用到的对象。

通常，应用程序服务器（Application Server）提供了应用程序可以使用的多种服务。在谈到分布式事务时，该服务就称作 XA Resource。当然，在应用程序可以使用 XA Resource 之前，首先要在应用程序服务器中注册和配置 XA Resource。

事务管理器作为管理和协调分布式事务的关键处理中心非常重要，所以应用服务器可以单独只用过事务管理器。

## 2.在SpringBoot中使用分布式事务

上面主要是一些基本的概念，在学习研究中总结出来的，可能不太全面，下面主要介绍一下在使用Spring使用分布式事务中的心得，这种做法也是将事务管理器嵌入应用中。

开始准备Spring的时候，Spring官网-SpringBoot文档第38章介绍了Atomikos和Bitronix 等工具，实际上这些工具都是取代应用服务器对事务管理器的支持，负责实现事务管理器对象。由于Atomikos介绍在Bitronix 之前，所以直接使用Atomikos进行测试。

### 2.1.盲点解释

要理解 JTA 的实现原理首先需要了解其架构：它包括事务管理器（Transaction Manager）和一个或多个支持 XA 协议的资源管理器 ( Resource Manager ) 两部分， 我们可以将资源管理器看做任意类型的持久化数据存储；事务管理器则承担着所有事务参与单元的协调与控制。 根据所面向对象的不同，我们可以将 JTA 的事务管理器和资源管理器理解为两个方面：面向开发人员的使用接口（事务管理器）和面向服务提供商的实现接口（资源管理器）。其中开发接口的主要部分即为上述示例中引用的 UserTransaction 对象，开发人员通过此接口在信息系统中实现分布式事务；而实现接口则用来规范提供商（如数据库连接提供商）所提供的事务服务，它约定了事务的资源管理功能，使得 JTA 可以在异构事务资源之间执行协同沟通。以数据库为例，IBM 公司提供了实现分布式事务的数据库驱动程序，Oracle 也提供了实现分布式事务的数据库驱动程序， 在同时使用 DB2 和 Oracle 两种数据库连接时， JTA 即可以根据约定的接口协调者两种事务资源从而实现分布式事务。正是基于统一规范的不同实现使得 JTA 可以协调与控制不同数据库或者 JMS 厂商的事务资源，其架构如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205164907157.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)
图 2.1 JTA体系结构

开发人员使用开发人员接口，实现应用程序对全局事务的支持；各提供商（数据库，JMS 等）依据提供商接口的规范提供事务资源管理功能；事务管理器（ TransactionManager ）将应用对分布式事务的使用映射到实际的事务资源并在事务资源间进行协调与控制。 下面，本文将对包括 UserTransaction、Transaction 和 TransactionManager 在内的三个主要接口以及其定义的方法进行介绍。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205164937608.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

- UserTransaction:面向开发人员的接口,开发人员通常只使用此接口实现 JTA 事务管理
- Transaction:代表了一个物理意义上的事务
- TransactionManager:本身并不承担实际的事务处理功能，它更多的是充当用户接口和实现接口之间的桥梁
- UserTransaction 对象不会对事务进行任何控制，所有的事务方法都是通过 TransactionManager 传递到实际的事务资源即 Transaction 对象上

### 2.2.确保mysql开启XA事务支持

```sql
SHOW VARIABLES LIKE '%xa%'  
```

如果innodb_support_xa的值是ON就说明mysql已经开启对XA事务的支持了。 
如果不是就执行：

```sql
SET innodb_support_xa = ON  
```

### 2.3.重要Maven依赖

```xml
<!-- 数据库连接 -->    
<dependency>    
    <groupId>mysql</groupId>    
    <artifactId>mysql-connector-java</artifactId>    
    <scope>runtime</scope>    
</dependency>    
<!-- 分布式事务atomikos -->    
<dependency>    
    <groupId>org.springframework.boot</groupId>    
    <artifactId>spring-boot-starter-jta-atomikos</artifactId>    
</dependency>    
<!-- Druid -->  
<dependency>  
    <groupId>com.alibaba</groupId>  
    <artifactId>druid-spring-boot-starter</artifactId>  
    <version>1.1.10</version>  
</dependency>  
<!-- MyBatis -->  
<dependency>  
    <groupId>org.mybatis.spring.boot</groupId>  
    <artifactId>mybatis-spring-boot-starter</artifactId>  
    <version>1.3.2</version>  
</dependency>  
```

### 2.4.配置Atomikos, Druid, MyBatis

首先，要使下面的代码配置生效要先确保你在项目工程中引入了spring-boot-starter-jta-atomikos, druid-spring-boot-starter这两个依赖。
第二，SpringBoot会自动配置Atomikos的事务管理配置，无需做其它的配置。

#### 2.4.1.application.properties

第15-19行代码表示实现javax.sql.XADataSource接口的com.alibaba.druid.pool.xa.DruidXADataSource的特有属性, 并不是Atomikos的属性.

```yaml
spring.application.name=learn-jta-atomikos

# 开启下划线-驼峰命名转换
mybatis.configuration.map-underscore-to-camel-case=true

spring.aop.proxy-target-class=true

## jta相关参数配置
# 如果你在JTA环境中，并且仍然希望使用本地事务，你可以设置spring.jta.enabled属性为false以禁用JTA自动配置。
spring.jta.enabled=true
# 必须配置唯一的资源名
spring.jta.atomikos.datasource.one.unique-resource-name=jta-personal
# 配置Druid的属性 https://github.com/alibaba/druid/wiki/DruidDataSource%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8
spring.jta.atomikos.datasource.one.xa-data-source-class-name=com.alibaba.druid.pool.xa.DruidXADataSource
spring.jta.atomikos.datasource.one.xa-properties.url=jdbc:mysql://localhost:3306/personal?characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
spring.jta.atomikos.datasource.one.xa-properties.username=root
spring.jta.atomikos.datasource.one.xa-properties.password=123456
spring.jta.atomikos.datasource.one.xa-properties.filters=slf4j,stat,wall,config
#spring.jta.atomikos.datasource.one.xa-properties.connectionProperties=config.decrypt=true;config.decrypt.key=${druid.publickey}

spring.jta.atomikos.datasource.two.unique-resource-name=jta-book
spring.jta.atomikos.datasource.two.max-pool-size=8
spring.jta.atomikos.datasource.two.xa-data-source-class-name=com.alibaba.druid.pool.xa.DruidXADataSource
spring.jta.atomikos.datasource.two.xa-properties.url=jdbc:mysql://localhost:3306/secondary?characterEncoding=utf-8&useSSL=false&&allowMultiQueries=true
spring.jta.atomikos.datasource.two.xa-properties.username=root
spring.jta.atomikos.datasource.two.xa-properties.password=123456
spring.jta.atomikos.datasource.two.xa-properties.filters=slf4j,stat,wall,config
#spring.jta.atomikos.datasource.two.xa-properties.connectionProperties=config.decrypt=true;config.decrypt.key=${druid.publickey}

## Druid监控设置
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=true
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=admin
spring.datasource.druid.aop-patterns=com.example.atomikos.service.*
```

#### 2.4.2.配置Atomikos数据源与MyBatis集成

这里只给出默认数据源的Atomikos与MyBatis的集成，其余的数据源的配置与它大同小异(见第2.5章实例)。
注意！第18行代码，这里指定com.example.atomikos.dao.one这个包路径下Mapper接口的MyBatis的会话工厂，不同的数据源指定不同的会话工厂！！！
然后在使用dao层的时候，正常使用即可，详细代码见(见第2.5章实例)。

```java
package com.example.atomikos.config;

import com.atomikos.jdbc.AtomikosDataSourceBean;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

/**
 * 设置JTA(Atomikos)数据源，Mybatis
 *
 * @author fengxuechao
 */
@Configuration
@MapperScan(basePackages = "com.example.atomikos.dao.one", sqlSessionFactoryRef = "oneSqlSessionFactory")
public class OneDatabaseConfig {

    /**
     * 设置JTA(Atomikos)数据源
     *
     * @return {@link AtomikosDataSourceBean}
     */
    @Primary
    @Bean
    @ConfigurationProperties(prefix = "spring.jta.atomikos.datasource.one")
    public DataSource oneDataSource() {
        return new AtomikosDataSourceBean();
    }

    /**
     * 设置Mybatis的会话工厂类
     *
     * @param dataSource JTA(Atomikos)数据源
     * @return {@link SqlSessionFactoryBean#getObject()}
     * @throws Exception
     */
    @Primary
    @Bean(name = "oneSqlSessionFactory")
    public SqlSessionFactory oneSqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        return bean.getObject();
    }
}
```

#### 2.4.3.配置声明式事务(tx+aop)

Java配置声明式事务AOP

```java
import org.springframework.aop.Advisor;  
import org.springframework.aop.aspectj.AspectJExpressionPointcut;  
import org.springframework.aop.support.DefaultPointcutAdvisor;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.transaction.PlatformTransactionManager;  
import org.springframework.transaction.TransactionDefinition;  
import org.springframework.transaction.interceptor.*;  
  
import java.util.Collections;  
import java.util.HashMap;  
import java.util.Map;  
  
/** 
 * 配置声明式事务 切面拦截 
 * 
 * @author fengxuechao 
 */  
@Configuration  
public class TransactionConfig {  
  
    private static final int TX_METHOD_TIMEOUT = 5;  
    private static final String AOP_POINTCUT_EXPRESSION = "execution (* com.example.atomikos.service.*.*(..))";  
  
    @Autowired  
    private PlatformTransactionManager transactionManager;  
  
    @Bean  
    public TransactionInterceptor txAdvice() {  
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();  
  
        /* 只读事务，不做更新操作 */  
        RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();  
        readOnlyTx.setReadOnly(true);  
        readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);  
  
        /* 当前存在事务就使用当前事务，当前不存在事务就创建一个新的事务 */  
        RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();  
        requiredTx.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class)));  
        requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);  
        requiredTx.setTimeout(TX_METHOD_TIMEOUT);  
        Map<String, TransactionAttribute> txMap = new HashMap<>(10);  
  
        txMap.put("add*", requiredTx);  
        txMap.put("save*", requiredTx);  
        txMap.put("insert*", requiredTx);  
        txMap.put("update*", requiredTx);  
        txMap.put("delete*", requiredTx);  
  
        txMap.put("get*", readOnlyTx);  
        txMap.put("query*", readOnlyTx);  
        txMap.put("list*", readOnlyTx);  
        txMap.put("find*", readOnlyTx);  
        source.setNameMap(txMap);  
        return new TransactionInterceptor(transactionManager, source);  
    }  
  
    /** 
     * 切点 
     * 
     * @return 
     */  
    @Bean  
    public Advisor txAdviceAdvisor() {  
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();  
        pointcut.setExpression(AOP_POINTCUT_EXPRESSION);  
        return new DefaultPointcutAdvisor(pointcut, txAdvice());  
    }  
  
}  
```

等同于下面的Spring XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:tx="http://www.springframework.org/schema/tx"  
       xmlns:aop="http://www.springframework.org/schema/aop"  
       xsi:schemaLocation="  
       http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd  
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd  
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">  
  
    <!-- 配置事务传播特性 -->  
    <tx:advice id="txAdvice">  
        <tx:attributes>  
            <!--  
                            name        ：绑定事务的方法名，可以使用通配符，可以配置多个。  
                            propagation ：传播行为  
                            isolation   ：隔离级别  
                            read-only   ：是否只读  
                            timeout     ：超时信息  
                            rollback-for：发生哪些异常回滚.  
                            no-rollback-for：发生哪些异常不回滚.  
                        -->  
            <!-- 哪些方法加事务 -->  
            <tx:method name="query*" propagation="SUPPORTS" read-only="true"/>  
            <tx:method name="get*" propagation="SUPPORTS" read-only="true"/>  
            <tx:method name="select*" propagation="SUPPORTS" read-only="true"/>  
            <tx:method name="list*" propagation="SUPPORTS" read-only="true"/>  
            <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>  
            <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>  
            <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>  
            <tx:method name="add*" propagation="REQUIRED" rollback-for="Exception"/>  
        </tx:attributes>  
    </tx:advice>  
  
    <aop:config>  
        <!-- 注意：如果是自己编写的切面，使用<aop:aspect>标签，如果是系统制作的，使用<aop:advisor>标签。 -->  
        <aop:advisor advice-ref="txAdvice" pointcut="execution (* com.example.atomikos.service.*.*(..))" order="0" />  
    </aop:config>  
</beans>  
```

### 2.5.实例

#### 2.5.1.项目结构

![项目工程结构](https://img-blog.csdnimg.cn/2018120517000116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

#### 2.5.2.数据库

personal.sql

```sql
create table if not exists article  
(  
  id bigint unsigned auto_increment  
    primary key,  
  content varchar(255) null,  
  title varchar(255) null,  
  url varchar(255) null  
);  
  
create table if not exists user  
(  
  id bigint unsigned auto_increment primary key,  
  username varchar(255) charset utf8 null,  
  password varchar(255) charset utf8 null  
);  
```

secondary.sql

```sql
create table book  
(  
  id         bigint unsigned auto_increment primary key,  
  name       varchar(255)    null,  
  article_id bigint unsigned null,  
  user_id    bigint unsigned null  
);  
```

#### 2.5.3.Maven依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
  
    <groupId>com.example</groupId>  
    <artifactId>learn-jta-atomikos-SpringBoot</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <packaging>jar</packaging>  
  
    <name>learn-jta-atomikos</name>  
    <description>Demo project for Spring Boot</description>  
  
    <parent>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-parent</artifactId>  
        <version>2.0.5.RELEASE</version>  
        <relativePath/> <!-- lookup parent from repository -->  
    </parent>  
  
    <properties>  
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>  
        <java.version>1.8</java.version>  
    </properties>  
  
    <dependencies>  
        <!-- MyBatis -->  
        <dependency>  
            <groupId>org.mybatis.spring.boot</groupId>  
            <artifactId>mybatis-spring-boot-starter</artifactId>  
            <version>1.3.2</version>  
            <exclusions>  
                <exclusion>  
                    <artifactId>spring-boot-starter-logging</artifactId>  
                    <groupId>org.springframework.boot</groupId>  
                </exclusion>  
            </exclusions>  
        </dependency>  
        <!-- 热部署 -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-devtools</artifactId>  
            <scope>runtime</scope>  
        </dependency>  
        <!-- 数据库连接 -->  
        <dependency>  
            <groupId>mysql</groupId>  
            <artifactId>mysql-connector-java</artifactId>  
            <scope>runtime</scope>  
        </dependency>  
        <dependency>  
            <groupId>com.alibaba</groupId>  
            <artifactId>druid-spring-boot-starter</artifactId>  
            <version>1.1.10</version>  
        </dependency>  
        <!-- 分布式事务atomikos -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-jta-atomikos</artifactId>  
        </dependency>  
        <!-- tx + aop -->  
        <dependency>  
            <groupId>org.aspectj</groupId>  
            <artifactId>aspectjweaver</artifactId>  
            <version>1.9.2</version>  
        </dependency>  
        <!-- 添加Log4j2 -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-log4j2</artifactId>  
        </dependency>  
        <!-- 为log4j2添加异步支持 -->  
        <dependency>  
            <groupId>com.lmax</groupId>  
            <artifactId>disruptor</artifactId>  
            <version>3.4.2</version>  
        </dependency>  
        <!-- 简化代码 -->  
        <dependency>  
            <groupId>org.projectlombok</groupId>  
            <artifactId>lombok</artifactId>  
        </dependency>  
        <!-- 用于监控与管理 -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-actuator</artifactId>  
            <exclusions>  
                <exclusion>  
                    <artifactId>spring-boot-starter-logging</artifactId>  
                    <groupId>org.springframework.boot</groupId>  
                </exclusion>  
            </exclusions>  
        </dependency>  
        <!-- WEB -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-web</artifactId>  
        </dependency>  
        <!-- 配合@ConfigurationProperties编译生成元数据文件(IDEA编辑器的属性提示) -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-configuration-processor</artifactId>  
            <optional>true</optional>  
        </dependency>  
        <!-- 测试 -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-test</artifactId>  
            <scope>test</scope>  
        </dependency>  
    </dependencies>  
  
    <build>  
        <plugins>  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
            </plugin>  
        </plugins>  
    </build>  
</project>  
```

#### 2.5.4.配置

##### application.properties

```yaml
spring.application.name=learn-jta-atomikos  
  
# 开启下划线-驼峰命名转换  
mybatis.configuration.map-underscore-to-camel-case=true  
  
spring.aop.proxy-target-class=true  
  
## jta相关参数配置  
# 如果你在JTA环境中，并且仍然希望使用本地事务，你可以设置spring.jta.enabled属性为false以禁用JTA自动配置。  
spring.jta.enabled=true  
# 必须配置唯一的资源名  
spring.jta.atomikos.datasource.one.unique-resource-name=jta-personal  
# 配置Druid的属性 https://github.com/alibaba/druid/wiki/DruidDataSource%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8  
spring.jta.atomikos.datasource.one.xa-data-source-class-name=com.alibaba.druid.pool.xa.DruidXADataSource  
spring.jta.atomikos.datasource.one.xa-properties.url=jdbc:mysql://localhost:3306/personal?characterEncoding=utf-8&useSSL=false&allowMultiQueries=true  
spring.jta.atomikos.datasource.one.xa-properties.username=root  
spring.jta.atomikos.datasource.one.xa-properties.password=123456  
spring.jta.atomikos.datasource.one.xa-properties.filters=slf4j,stat,wall,config  
#spring.jta.atomikos.datasource.one.xa-properties.connectionProperties=config.decrypt=true;config.decrypt.key=${druid.publickey}  
  
spring.jta.atomikos.datasource.two.unique-resource-name=jta-book  
spring.jta.atomikos.datasource.two.max-pool-size=8  
spring.jta.atomikos.datasource.two.xa-data-source-class-name=com.alibaba.druid.pool.xa.DruidXADataSource  
spring.jta.atomikos.datasource.two.xa-properties.url=jdbc:mysql://localhost:3306/secondary?characterEncoding=utf-8&useSSL=false&&allowMultiQueries=true  
spring.jta.atomikos.datasource.two.xa-properties.username=root  
spring.jta.atomikos.datasource.two.xa-properties.password=123456  
spring.jta.atomikos.datasource.two.xa-properties.filters=slf4j,stat,wall,config  
#spring.jta.atomikos.datasource.two.xa-properties.connectionProperties=config.decrypt=true;config.decrypt.key=${druid.publickey}  
  
## Druid监控设置  
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*  
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*  
spring.datasource.druid.stat-view-servlet.reset-enable=true  
spring.datasource.druid.stat-view-servlet.login-username=admin  
spring.datasource.druid.stat-view-servlet.login-password=admin  
spring.datasource.druid.aop-patterns=com.example.atomikos.service.*  
```

##### OneDatabaseConfig

```java
package com.example.atomikos.config;  
  
import com.atomikos.jdbc.AtomikosDataSourceBean;  
import org.apache.ibatis.session.SqlSessionFactory;  
import org.mybatis.spring.SqlSessionFactoryBean;  
import org.mybatis.spring.annotation.MapperScan;  
import org.springframework.boot.context.properties.ConfigurationProperties;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.Primary;  
  
import javax.sql.DataSource;  
  
/** 
 * 设置JTA(Atomikos)数据源，Mybatis 
 * 
 * @author fengxuechao 
 */  
@Configuration  
@MapperScan(basePackages = "com.example.atomikos.dao.one", sqlSessionFactoryRef = "oneSqlSessionFactory")  
public class OneDatabaseConfig {  
  
    /** 
     * 设置JTA(Atomikos)数据源 
     * 
     * @return {@link AtomikosDataSourceBean} 
     */  
    @Primary  
    @Bean  
    @ConfigurationProperties(prefix = "spring.jta.atomikos.datasource.one")  
    public DataSource oneDataSource() {  
        return new AtomikosDataSourceBean();  
    }  
  
    /** 
     * 设置Mybatis的会话工厂类 
     * 
     * @param dataSource JTA(Atomikos)数据源 
     * @return {@link SqlSessionFactoryBean#getObject()} 
     * @throws Exception 
     */  
    @Primary  
    @Bean(name = "oneSqlSessionFactory")  
    public SqlSessionFactory oneSqlSessionFactory(DataSource dataSource) throws Exception {  
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();  
        bean.setDataSource(dataSource);  
        return bean.getObject();  
    }  
}  
```

##### TwoDatabaseConfig 

```java
package com.example.atomikos.config;  
  
import com.atomikos.jdbc.AtomikosDataSourceBean;  
import org.apache.ibatis.session.SqlSessionFactory;  
import org.mybatis.spring.SqlSessionFactoryBean;  
import org.mybatis.spring.annotation.MapperScan;  
import org.springframework.beans.factory.annotation.Qualifier;  
import org.springframework.boot.context.properties.ConfigurationProperties;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
import javax.sql.DataSource;  
  
/** 
 * 设置JTA(Atomikos)数据源，Mybatis 
 * 
 * @author fengxuechao 
 */  
@Configuration  
@MapperScan(basePackages = "com.example.atomikos.dao.two", sqlSessionFactoryRef = "twoSqlSessionFactory")  
public class TwoDatabaseConfig {  
  
    /** 
     * 设置JTA(Atomikos)数据源 
     * 
     * @return {@link AtomikosDataSourceBean} 
     */  
    @Bean(name = "twoAtomikosDataSource")  
    @ConfigurationProperties(prefix = "spring.jta.atomikos.datasource.two")  
    public DataSource oneDataSource() {  
        return new AtomikosDataSourceBean();  
    }  
  
    /** 
     * 设置Mybatis的会话工厂类 
     * 
     * @param dataSource JTA(Atomikos)数据源 
     * @return {@link SqlSessionFactoryBean#getObject()} 
     * @throws Exception 
     */  
    @Bean(name = "twoSqlSessionFactory")  
    public SqlSessionFactory oneSqlSessionFactory(@Qualifier("twoAtomikosDataSource") DataSource dataSource) throws Exception {  
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();  
        bean.setDataSource(dataSource);  
        return bean.getObject();  
    }  
}  
```

##### TransactionConfig 

```java
package com.example.atomikos.config;  
  
import org.springframework.aop.Advisor;  
import org.springframework.aop.aspectj.AspectJExpressionPointcut;  
import org.springframework.aop.support.DefaultPointcutAdvisor;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.transaction.PlatformTransactionManager;  
import org.springframework.transaction.TransactionDefinition;  
import org.springframework.transaction.interceptor.*;  
  
import java.util.Collections;  
import java.util.HashMap;  
import java.util.Map;  
  
/** 
 * 配置声明式事务 切面拦截 
 * 
 * @author fengxuechao 
 */  
@Configuration  
public class TransactionConfig {  
  
    private static final int TX_METHOD_TIMEOUT = 5;  
    private static final String AOP_POINTCUT_EXPRESSION = "execution (* com.example.atomikos.service.*.*(..))";  
  
    @Autowired  
    private PlatformTransactionManager transactionManager;  
  
    @Bean  
    public TransactionInterceptor txAdvice() {  
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();  
  
        /* 只读事务，不做更新操作 */  
        RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();  
        readOnlyTx.setReadOnly(true);  
        readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);  
  
        /* 当前存在事务就使用当前事务，当前不存在事务就创建一个新的事务 */  
        RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();  
        requiredTx.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class)));  
        requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);  
        requiredTx.setTimeout(TX_METHOD_TIMEOUT);  
        Map<String, TransactionAttribute> txMap = new HashMap<>(10);  
  
        txMap.put("add*", requiredTx);  
        txMap.put("save*", requiredTx);  
        txMap.put("insert*", requiredTx);  
        txMap.put("update*", requiredTx);  
        txMap.put("delete*", requiredTx);  
  
        txMap.put("get*", readOnlyTx);  
        txMap.put("query*", readOnlyTx);  
        txMap.put("list*", readOnlyTx);  
        txMap.put("find*", readOnlyTx);  
        source.setNameMap(txMap);  
        return new TransactionInterceptor(transactionManager, source);  
    }  
  
    /** 
     * 切点 
     * 
     * @return 
     */  
    @Bean  
    public Advisor txAdviceAdvisor() {  
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();  
        pointcut.setExpression(AOP_POINTCUT_EXPRESSION);  
        return new DefaultPointcutAdvisor(pointcut, txAdvice());  
    }  
  
}  
```

#### 2.5.5.实体类

```java
package com.example.atomikos.entity;  
  
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
  
import java.io.Serializable;  
  
/** 
 * 文章 
 * 
 * @author fengxuechao 
 */  
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class ArticleDO implements Serializable {  
  
    private static final long serialVersionUID = 3971756585655871603L;  
  
    private Long id;  
  
    private String title;  
  
    private String content;  
  
    private String url;  
  
}  
```

```java
package com.example.atomikos.entity;  
  
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
  
import java.io.Serializable;  
  
/** 
 * 书 
 * 
 * @author fengxuechao 
 */  
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class BookDO implements Serializable {  
  
    private static final long serialVersionUID = 3231762613546697469L;  
  
    private Long id;  
  
    private String name;  
  
    private Long articleId;  
  
    private Long userId;  
  
}  
```

```java
package com.example.atomikos.entity;  
  
import lombok.Data;  
  
@Data  
public class BookVo extends BookDO {  
  
    private UserDO user;  
}  
 ```
 
 ```java
package com.example.atomikos.entity;  
  
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
  
import java.io.Serializable;  
  
/** 
 * 用户 
 * 
 * @author fengxuechao 
 */  
@Data  
@NoArgsConstructor  
@AllArgsConstructor  
public class UserDO implements Serializable {  
  
    private static final long serialVersionUID = 469663920369239035L;  
  
    private Long id;  
  
    private String username;  
  
    private String password;  
}  
```

#### 2.5.6.Dao层

##### UserDao

注意包名，UserDao对应的配置为OneDatabaseConfig

```java
package com.example.atomikos.dao.one;  
  
import com.example.atomikos.entity.UserDO;  
import org.apache.ibatis.annotations.*;  
import org.springframework.stereotype.Repository;  
  
import java.util.List;  
  
/** 
 * @author fengxuechao 
 * @date 2018/11/30 
 */  
@Repository  
public interface UserDao {  
  
    /** 
     * 根据主键查询一条记录 
     * 
     * @param id 
     * @return 
     */  
    @Select("select id, username, password from user where id = #{id}")  
    UserDO get(Long id);  
  
    /** 
     * 分页列表查询 
     * 
     * @param page 
     * @param size 
     * @return 
     */  
    @Select("select id, username, password from user limit #{page}, #{size}")  
    List<UserDO> list(Integer page, Integer size);  
  
    /** 
     * 保存 
     * 
     * @param userDO 
     * @return 自增主键 
     */  
    @Insert("insert into user(username, password) values(#{username}, #{password})")  
    @Options(useGeneratedKeys = true, keyColumn = "id")  
    int save(UserDO userDO);  
  
    /** 
     * 修改一条记录 
     * 
     * @param user 
     * @return 
     */  
    @Update("update user set username = #{username}, password = #{password} where id = #{id}")  
    int update(UserDO user);  
  
    /** 
     * 删除一条记录 
     * 
     * @param id 主键 
     * @return 
     */  
    @Delete("delete from user where id = #{id}")  
    int delete(Long id);  
}  
```

##### BookDao

注意包名，UserDao对应的配置为TwoDatabaseConfig

```java
package com.example.atomikos.dao.two;  
  
import com.example.atomikos.entity.BookDO;  
import org.apache.ibatis.annotations.*;  
import org.springframework.stereotype.Repository;  
  
import java.util.List;  
  
/** 
 * @author fengxuechao 
 * @date 2018/11/30 
 */  
@Mapper  
@Repository  
public interface BookDao {  
  
    /** 
     * 分页查询 
     * 
     * @param page 页码 
     * @param size 每页记录数 
     * @return 
     */  
    @Select("select id, name, article_id as articleId, user_id as userId from book limit ${page}, ${size}")  
    List<BookDO> list(@Param("page") Integer page, @Param("size") Integer size);  
  
    /** 
     * 根据主键查询单条记录 
     * 
     * @param id 
     * @return 
     */  
    @Select("select id, name, article_id as articleId, user_id as userId from book where id = #{id}")  
    BookDO get(Long id);  
  
    /** 
     * 添加一条记录 
     * 
     * @param book 
     * @return 自增主键 
     */  
    @Insert("insert into book(name, article_id, user_id) values(#{name}, #{articleId}, #{userId})")  
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")  
    int save(BookDO book);  
  
    /** 
     * 修改一条记录 
     * 
     * @param book 
     * @return 
     */  
    @Update("update book set name = #{name}, article_id = #{articleId}, user_id = #{userId} where id = #{id}")  
    int update(BookDO book);  
  
    /** 
     * 删除一条记录 
     * 
     * @param id 主键 
     * @return 
     */  
    @Delete("delete from book where id = #{id}")  
    int delete(Long id);  
}  
```

#### 2.5.7.Service层

##### Bookservice

```java
package com.example.atomikos.service;  
  
import com.example.atomikos.entity.BookDO;  
import com.example.atomikos.entity.UserDO;  
  
import java.util.List;  
  
/** 
 * 主要目的是测试分布式事务 
 * 
 * @author fengxuechao 
 */  
public interface BookService {  
  
    /** 
     * 保存 
     * 
     * @param book 
     * @param user 
     * @return 
     */  
    BookDO save(BookDO book, UserDO user);  
  
    /** 
     * 单条查询 
     * 
     * @param id 
     * @return 
     */  
    BookDO get(Long id);  
  
    /** 
     * 分页查询 
     * 
     * @param page 
     * @param size 
     * @return 
     */  
    List<BookDO> list(Integer page, Integer size);  
  
}  
```

##### BookServiceImpl

请注意，其中有些代码故意抛出异常是为了测试的目的。

```java
package com.example.atomikos.service.impl;  
  
import com.example.atomikos.dao.one.UserDao;  
import com.example.atomikos.dao.two.BookDao;  
import com.example.atomikos.entity.BookDO;  
import com.example.atomikos.entity.UserDO;  
import com.example.atomikos.service.BookService;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Service;  
import org.springframework.transaction.annotation.Transactional;  
  
import java.util.List;  
  
/** 
 * @author fengxuechao 
 */  
@Service  
public class BookServiceImpl implements BookService {  
  
    @Autowired  
    private BookDao bookDao;  
  
    @Autowired  
    private UserDao userDao;  
  
    /** 
     * 保存书本和文章, 使用声明式事务(tx+aop形式) 
     * 
     * @param book {@link BookDO} 
     * @param user {@link UserDO} 
     * @return 
     */  
    @Override  
    public BookDO save(BookDO book, UserDO user) {  
        int userSave = userDao.save(user);  
        if (userSave == 0) {  
            return null;  
        }  
        book.setUserId(user.getId());  
        int bookSave = bookDao.save(book);  
        if (bookSave == 0) {  
            return null;  
        }  
//        throw new RuntimeException("测试分布式事务(tx+aop形式)");  
        return book;  
    }  
  
    /** 
     * 单条查询 
     * 
     * @param id 
     * @return 
     */  
    @Override  
    public BookDO get(Long id) {  
        BookDO book = bookDao.get(id);  
        UserDO user = userDao.get(book.getUserId());  
        return new BookDO(book.getId(), book.getName(), book.getArticleId(), user.getId());  
    }  
  
    /** 
     * 分页查询 
     * 
     * @param page 
     * @param size 
     * @return 
     */  
    @Override  
    public List<BookDO> list(Integer page, Integer size) {  
        page = (page < 1 ? 0 : page - 1) * size;  
        return bookDao.list(page, size);  
    }  
  
    /** 
     * 修改书本和文章, 使用声明式事务(注解形式) 
     * 
     * @param book 
     * @param user 
     * @return 
     */  
    @Transactional(rollbackFor = Exception.class)  
    public BookDO update(BookDO book, UserDO user) {  
        int bookUpdate = bookDao.update(book);  
        if (bookUpdate != 1) {  
            return null;  
        }  
        int userUpdate = userDao.update(user);  
        if (userUpdate != 1) {  
            return null;  
        }  
        throw new RuntimeException("测试分布式事务(注解形式)");  
//        return book;  
    }  
  
    /** 
     * 删除书本和文章 
     * 
     * @param id 
     * @return 
     */  
    public int delete(Long id) {  
        BookDO book = bookDao.get(id);  
        System.err.println(book);  
        Long userId = book.getUserId();  
        int userDelete = userDao.delete(userId);  
        if (userDelete != 1) {  
            return 0;  
        }  
        int bookDelete = bookDao.delete(id);  
        if (bookDelete != 1) {  
            return 0;  
        }  
        throw new RuntimeException("测试没有添加分布式事务管理)");  
//        return 1;  
    }  
  
}  
```

#### 2.5.8.Controller层

```java
package com.example.atomikos.controller;  
  
import com.example.atomikos.entity.BookDO;  
import com.example.atomikos.entity.BookVo;  
import com.example.atomikos.service.BookService;  
import com.example.atomikos.service.impl.BookServiceImpl;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.web.bind.annotation.*;  
  
import java.util.List;  
  
/** 
 * @author fengxuechao 
 */  
@RestController  
@RequestMapping("/books")  
public class BookController {  
  
    @Autowired  
    private BookService bookService;  
  
    @GetMapping  
    public List<BookDO> list(  
            @RequestParam(defaultValue = "1") Integer page,  
            @RequestParam(defaultValue = "10") Integer size) {  
        return bookService.list(page, size);  
    }  
  
    @GetMapping("/{id}")  
    public BookDO get(@PathVariable Long id) {  
        return bookService.get(id);  
    }  
  
    @PostMapping  
    public BookDO save(@RequestBody BookVo book) {  
        return bookService.save(book, book.getUser());  
    }  
  
    @PutMapping  
    public BookDO update(@RequestBody BookVo book) {  
        return ((BookServiceImpl) bookService).update(book, book.getUser());  
    }  
  
    @DeleteMapping("/{id}")  
    public int delete(@PathVariable Long id) {  
        return ((BookServiceImpl) bookService).delete(id);  
    }  
  
}  
```

#### 2.5.9.单元测试

##### BookServiceImplTest 

由于故意抛出异常，故单元测试失败，查看数据库，数据库中数据保持原样

```java
package com.example.atomikos.service.impl;  
  
import com.example.atomikos.entity.BookDO;  
import com.example.atomikos.entity.UserDO;  
import com.example.atomikos.service.BookService;  
import org.junit.Assert;  
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.boot.test.context.SpringBootTest;  
import org.springframework.test.context.junit4.SpringRunner;  
  
/** 
 * 测试分布式事务：切面拦截形式, 注解式 
 */  
@RunWith(SpringRunner.class)  
@SpringBootTest  
public class BookServiceImplTest {  
  
    @Autowired  
    BookService bookService;  
  
    /** 
     * 测试分布式事务(切面拦截形式) 
     */  
    @Test  
    public void save() {  
        BookDO book = new BookDO();  
        book.setName("Book Name - 001");  
        book.setArticleId(69L);  
  
        UserDO user = new UserDO();  
        user.setUsername("username - 001");  
        user.setPassword("password - 001");  
        BookDO bookDO = bookService.save(book, user);  
        System.out.println(bookDO);  
    }  
  
    /** 
     * 测试分布式事务(注解式) 
     */  
    @Test  
    public void update() {  
        BookDO book = new BookDO();  
        book.setId(10L);  
        book.setName("Book Name - 002");  
        book.setArticleId(69L);  
  
        UserDO user = new UserDO();  
        user.setId(18L);  
        user.setUsername("username - 002");  
        user.setPassword("password - 002");  
  
        ((BookServiceImpl)bookService).update(book, user);  
    }  
  
    /** 
     * 没有事务管理 
     */  
    @Test  
    public void delete() {  
        int delete = ((BookServiceImpl) bookService).delete(11L);  
        Assert.assertEquals(1, delete);  
    }  
}  
```

##### BookControllerTest 

```java
package com.example.atomikos.controller;  
  
import com.example.atomikos.dao.one.UserDao;  
import com.example.atomikos.entity.BookVo;  
import com.example.atomikos.entity.UserDO;  
import com.fasterxml.jackson.databind.ObjectMapper;  
import org.junit.Before;  
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.boot.test.context.SpringBootTest;  
import org.springframework.http.MediaType;  
import org.springframework.test.context.junit4.SpringRunner;  
import org.springframework.test.web.servlet.MockMvc;  
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;  
import org.springframework.test.web.servlet.setup.MockMvcBuilders;  
import org.springframework.web.context.WebApplicationContext;  
  
import static org.hamcrest.Matchers.is;  
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;  
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;  
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;  
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;  
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;  
  
/** 
 * 测试分布式事务 
 */  
@RunWith(SpringRunner.class)  
@SpringBootTest  
public class BookControllerTest {  
  
    private MockMvc mockMvc;  
  
    private ObjectMapper objectMapper = new ObjectMapper();  
  
    @Autowired  
    private UserDao userDao;  
  
    @Autowired  
    private WebApplicationContext context;  
  
    @Before  
    public void setUp() {  
        this.mockMvc = MockMvcBuilders.webAppContextSetup(this.context).build();  
    }  
  
    /** 
     * 申明式 
     * 
     * @throws Exception 
     */  
    @Test  
    public void save() throws Exception {  
        UserDO user = new UserDO();  
        user.setUsername("username - 002");  
        user.setPassword("password - 002");  
  
        BookVo book = new BookVo();  
        book.setName("Book Name - 002");  
        book.setArticleId(69L);  
        book.setUser(user);  
        String json = objectMapper.writeValueAsString(book);  
        this.mockMvc.perform(  
                post("/books")  
                        .contentType(MediaType.APPLICATION_JSON_UTF8)  
                        .content(json))  
                .andExpect(status().isOk())  
                .andExpect(jsonPath("$.name", is("Book Name - 002")))  
                .andExpect(jsonPath("$.articleId", is(69)))  
                .andDo(print());  
    }  
  
    /** 
     * 注解式 
     * 
     * @throws Exception 
     */  
    @Test  
    public void update() throws Exception {  
        UserDO user = userDao.get(3L);  
        assert user != null;  
        user.setUsername("username - 003");  
        user.setPassword("password - 003");  
  
        BookVo book = new BookVo();  
        book.setId(3L);  
        book.setName("Book Name - 003");  
        book.setArticleId(69L);  
        book.setUser(user);  
  
        String json = objectMapper.writeValueAsString(book);  
        this.mockMvc.perform(  
                put("/books")  
                        .contentType(MediaType.APPLICATION_JSON_UTF8)  
                        .content(json))  
                .andExpect(status().isOk())  
                .andExpect(jsonPath("$.name", is("Book Name - 003")))  
                .andExpect(jsonPath("$.articleId", is(69)))  
                .andDo(print());  
    }  
  
    /** 
     * 没有事务管理 
     * 
     * @throws Exception 
     */  
    @Test  
    public void delete() throws Exception {  
        this.mockMvc.perform(  
                MockMvcRequestBuilders.delete("/books/4"))  
                .andExpect(status().isOk())  
                .andDo(print());  
    }  
}  
```

## 3.引用

- [http://www.thedevpiece.com/configuring-multiple-datasources-using-springboot-and-atomikos/](http://www.thedevpiece.com/configuring-multiple-datasources-using-springboot-and-atomikos/)
- [https://blog.csdn.net/a510835147/article/details/75675311](https://blog.csdn.net/a510835147/article/details/75675311)
- [https://www.jianshu.com/p/0dde641295af](https://www.jianshu.com/p/0dde641295af)
- [http://www.importnew.com/15812.html](http://www.importnew.com/15812.html)
- [https://www.ibm.com/developerworks/cn/java/j-lo-jta/](https://www.ibm.com/developerworks/cn/java/j-lo-jta/)