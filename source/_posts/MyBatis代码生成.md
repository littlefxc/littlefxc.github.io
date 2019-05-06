---
title: MyBatis代码生成
date: 2019-05-06 19:09:33
categories: Java
tags:
- mybatis
---

要使用generator插件自动生成相关文件，需要引入mybatis-generator-core这个包，在 `<dependencys>` 中加入

<!-- more -->

## 1. 在pom.xml中做两处配置

### 1.1. 配置dependency

要使用generator插件自动生成相关文件，需要引入mybatis-generator-core这个包，在 `<dependencys>` 中加入：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### 1.2. 配置plugin

在 `<build>` 这个节点的 `<plugins>` 节点内部加入一个 `<plugin>`，如下：

```xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.5</version>
    <configuration>
        <configurationFile>${basedir}/src/main/resources/generator/generatorConfig.xml</configurationFile>
        <overwrite>true</overwrite>
        <verbose>true</verbose>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.5</version>
        </dependency>
    </dependencies>
</plugin>
```

## 2. 创建generatorConfig.xml

### 2.1. 配置文件路径名称以及内容

在resource目录下创建generatorConfig.xml配置文件，当然了该文件起这个名字，并且放到resource根目录下是根据genereator的默认方案来的，如果要用别的名，放到别的目录也可以，只是要做其它配置，这里就按默认算了，该文件的配置内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
"http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <!-- 数据库驱动:选择你的本地硬盘上面的数据库驱动包-->
    <classPathEntry location="E:\develop\mavenRepository\mysql\mysql-connector-java\5.1.46\mysql-connector-java-5.1.46.jar"/>
    <context id="mysqlTables" targetRuntime="MyBatis3">
        <!-- 自动识别数据库关键字，默认false，如果设置为true，根据SqlReservedWords中定义的关键字列表；
             一般保留默认值，遇到数据库关键字（Java关键字），使用columnOverride覆盖
        -->
        <property name="autoDelimitKeywords" value="false"/>
        <!-- 生成的Java文件的编码 -->
        <property name="javaFileEncoding" value="UTF-8"/>
        <!-- 格式化java代码 -->
        <property name="javaFormatter" value="org.mybatis.generator.api.dom.DefaultJavaFormatter"/>
        <!-- 格式化XML代码 -->
        <property name="xmlFormatter" value="org.mybatis.generator.api.dom.DefaultXmlFormatter"/>
        <!-- beginningDelimiter和endingDelimiter：指明数据库的用于标记数据库对象名的符号，比如ORACLE就是双引号，MYSQL默认是`反引号； -->
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <!--数据库链接URL，用户名、密码 -->
        <jdbcConnection
            driverClass="com.mysql.jdbc.Driver"
            connectionURL="jdbc:mysql://localhost:3306/personal?characterEncoding=UTF-8"
            userId="root"
            password="123456"></jdbcConnection>
        <!-- java类型处理器
             用于处理DB中的类型到Java中的类型，默认使用JavaTypeResolverDefaultImpl；
             注意一点，默认会先尝试使用Integer，Long，Short等来对应DECIMAL和 NUMERIC数据类型；
        -->
        <javaTypeResolver type="org.mybatis.generator.internal.types.JavaTypeResolverDefaultImpl">
        <!--
             true：使用BigDecimal对应DECIMAL和 NUMERIC数据类型
             false：默认,
             scale>0;length>18：使用BigDecimal;
             scale=0;length[10,18]：使用Long；
             scale=0;length[5,9]：使用Integer；
             scale=0;length<5：使用Short；
        -->
        <property name="forceBigDecimals" value="false"/></javaTypeResolver>
        <!-- 生成模型的包名和位置-->
        <javaModelGenerator targetPackage="com.littlefxc.personal.model" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>
        <!-- 生成映射文件的包名和位置-->
        <sqlMapGenerator targetPackage="mybatis" targetProject="src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
        <!-- 生成DAO的包名和位置-->
        <javaClientGenerator type="XMLMAPPER" targetPackage="com.littlefxc.personal.mapper" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <!-- 要生成的表 tableName是数据库中的表名或视图名 domainObjectName是实体类名-->
        <!-- generatedKey用于生成生成主键的方法，
             如果设置了该元素，MBG会在生成的<insert>元素中生成一条正确的<selectKey>元素，该元素可选
             column:主键的列名；
             sqlStatement：要生成的selectKey语句，有以下可选项：
             Cloudscape:相当于selectKey的SQL为： VALUES IDENTITY_VAL_LOCAL()
             DB2 :相当于selectKey的SQL为： VALUES IDENTITY_VAL_LOCAL()
             DB2_MF :相当于selectKey的SQL为：SELECT IDENTITY_VAL_LOCAL() FROM SYSIBM.SYSDUMMY1
             Derby :相当于selectKey的SQL为：VALUES IDENTITY_VAL_LOCAL()
             HSQLDB :相当于selectKey的SQL为：CALL IDENTITY()
             Informix :相当于selectKey的SQL为：select dbinfo('sqlca.sqlerrd1') from systables where tabid=1
             MySql :相当于selectKey的SQL为：SELECT LAST_INSERT_ID()
             SqlServer :相当于selectKey的SQL为：SELECT SCOPE_IDENTITY()
             SYBASE :相当于selectKey的SQL为：SELECT @@IDENTITY
             JDBC :相当于在生成的insert元素上添加useGeneratedKeys="true"和keyProperty属性
             <generatedKey column="" sqlStatement=""/>
        -->
        <table tableName="article"
            domainObjectName="ArticlePo"
            enableCountByExample="false"
            enableUpdateByExample="false"
            enableDeleteByExample="false"
            enableSelectByExample="false"
            selectByExampleQueryId="false">
            <generatedKey column="id" sqlStatement="MySql" identity="true"/>
        </table>

        <table tableName="article_tag"
            domainObjectName="ArticleTagPo"
            enableCountByExample="false"
            enableUpdateByExample="false"
            enableDeleteByExample="false"
            enableSelectByExample="false"
            selectByExampleQueryId="false">
            <generatedKey column="id" sqlStatement="MySql" identity="true"/>
        </table>

        <table tableName="user"
            domainObjectName="UserPo"
            enableCountByExample="false"
            enableUpdateByExample="false"
            enableDeleteByExample="false"
            enableSelectByExample="false"
            selectByExampleQueryId="false">
            <generatedKey column="id" sqlStatement="MySql" identity="true"/>
        </table>
    </context>
</generatorConfiguration>
 ```

## 3. 执行

### 3.1. 利用idea 执行  

![在这里插入图片描述](https://img-blog.csdn.net/2018102310013619?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 3.2. 也可以执行mvn mybatis-generator:generate命令

 生成最终效果图结构
![在这里插入图片描述](https://img-blog.csdn.net/2018102310014491?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)