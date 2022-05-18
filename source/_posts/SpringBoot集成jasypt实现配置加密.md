---
title: SpringBoot集成jasypt实现配置加密
date: 2021-06-02 19:32:08
categories: java/spring
tags:
- SpringBoot
- jasypt
- 加密
---

> 转载自 https://blog.lqdev.cn/2019/05/08/springboot/chapter-thirty-seven/

## 前言

> 近期在进行项目安全方面评审时，质量管理部门有提出需要对配置文件中的敏高文件进行加密处理，避免了信息泄露问题。想想前段时间某公司上传github时，把相应的生产数据库明文密码也一并上传了，导致了相应的数据泄露问题。也确实，大部分项目无论开发、测试还是生产环境，相关的敏高信息都是明文存储的，也是一大安全隐患呀。所以今天来说说，如何对配置文件进行加密操作。

## 一点知识

### 何为Jasypt

> [Jasypt](http://jasypt.org/)是一个Java库，允许开发人员以很简单的方式添加基本加密功能，而无需深入研究加密原理。利用它可以实现高安全性的，基于标准的加密技术，无论是单向和双向加密。加密密码，文本，数字，二进制文件。

1. 高安全性的，基于标准的加密技术，无论是单向和双向加密。加密密码，文本，数字，二进制文件…
2. 集成Hibernate的。
3. 可集成到Spring应用程序中，与Spring Security集成。
4. 集成的能力，用于加密的应用程序（即数据源）的配置。
5. 特定功能的高性能加密的multi-processor/multi-core系统。
6. 与任何JCE提供者使用开放的API

官网：http://www.jasypt.org/

## SpringBoot集成Jasypt

> `SpringBoot`中集成`Jasypt`，可直接使用开源的`jasypt-spring-boot`直接集成，使用简单方便。

![mark](https://gitee.com/littlefxc/oss/raw/master/images/WTvOjzNR7Xt5.png)

### 常规集成示例

1. 引入pom依赖

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>1.18</version>
</dependency>
```

2. 设置盐值和修改相应需要加密的配置参数

```
# 需要解密的地方，使用ENC()进行包裹处理
okong.name=ENC(Xj7Ykn2O0Hni/tN4oojPfw==)

# 设置盐值，生产环境中，切记不要直接进行设置，可通过环境变量、命令行等形式进行设置。
jasypt.encryptor.password=lqdev
```

简单来说，就是在需要加密的值使用`ENC(`和`)`进行包裹，即：`ENC(密文)`。若想避免参数冲突，可修改前缀和后缀，可以直接使用`jasypt.encryptor.property.prefix`和`jasypt.encryptor.property.suffix`进行修改即可。

**之后想往常一样使用`@Value("${}")`即可。**

### 包含xml引入时

> 在一些使用`javaBean`配置和`xml`两种混合模式时，使用第一种配置时，`xml`参数并未替换。此时看了官方文档，可以使用另一方式进行配置即可。

![官方说明](https://gitee.com/littlefxc/oss/raw/master/images/bPoPUpE8uk4A.png)

1. 引入pom依赖

    ```xml
    <dependency>
        <groupId>com.github.ulisesbocchio</groupId>
        <artifactId>jasypt-spring-boot</artifactId>
        <version>1.18</version>
    </dependency>
    ```

    其实就是不进行自动配置而已。

2. 启动类启动方式修改。

    ```java
    @SpringBootApplication
    @Slf4j
    public class JasyptApplication {
    
      public static void main(String[] args) throws Exception {
    
        // SpringApplication.run(JasyptApplication.class, args);
        // 使用自定义环境变量 实现一些特殊场景下的加密字符解密操作
        // 若无额外的xml引入文件需要解密时，可直接使用SpringApplication.run(JasyptApplication.class, args);即可
        // 若想在引入的xml中使用，需要加入环境变量，如以下模式
        new SpringApplicationBuilder().environment(new StandardEncryptableEnvironment())
        .sources(JasyptApplication.class).run(args);
        log.info("spring-boot-jasypt-chapter37服务启动!");
      }
    }
    ```

### 其他配置项

| Key                                     | Required | Default Value                       |
| --------------------------------------- | -------- | ----------------------------------- |
| jasypt.encryptor.password               | **True** | 盐值，根密码                        |
| jasypt.encryptor.algorithm              | False    | PBEWithMD5AndDES                    |
| jasypt.encryptor.keyObtentionIterations | False    | 1000                                |
| jasypt.encryptor.poolSize               | False    | 1                                   |
| jasypt.encryptor.providerName           | False    | SunJCE                              |
| jasypt.encryptor.providerClassName      | False    | null                                |
| jasypt.encryptor.saltGeneratorClassname | False    | org.jasypt.salt.RandomSaltGenerator |
| jasypt.encryptor.ivGeneratorClassname   | False    | org.jasypt.salt.NoOpIVGenerator     |
| jasypt.encryptor.stringOutputType       | False    | base64                              |
| jasypt.encryptor.proxyPropertySources   | False    | false                               |

## 运维说明

为了方便运维人员对各类敏感密钥进行加密操作，提供了自动化脚本，方便生成相应的加密串。

### 密钥（盐值）存储说明

本身加解密过程都是通过`盐值`进行处理的，所以正常情况下`盐值`和`加密串`是分开存储的。**`盐值`应该放在`系统属性`、`命令行`或是`环境变量`来使用，而不是放在配置文件。**

#### 命令行示例

```sh
java -jar xxx.jar --jasypt.encryptor.password=xxx &
```

#### 环境变量示例

设置环境变量：

```sh
vim /etc/profileexport JASYPT_PASSWORD = xxxx
```

启动命令：

```sh
java -jar xxx.jar --jasypt.encryptor.password=${JASYPT_PASSWORD} &
```

### bat脚本

为了方便，简单编写了一个bat脚本方便使用。

```bat
@echo off
set/p input=待加密的明文字符串：
set/p password=加密密钥(盐值)：
echo 加密中......
java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI  input=%input% password=%password% algorithm=PBEWithMD5AndDES
pause
```

**注意：`jasypt-1.9.2.jar` 文件需要和bat脚本放在相同目录下。此包可直接在示例项目中直接下载。**

使用示例：

**注意：相应加密串，每次加密的结果是不同的。**

![使用示例](https://gitee.com/littlefxc/oss/raw/master/images/568wfkAWA5Tl.png)

## 总结

本章节主要简单介绍了如何使用`jasypt`对配置文件进行加密操作。一些其他高级应用，可以查看官方文档进行相关集成即可。集成起来相对来说比较简单，注意是要对`密码(盐值)`的管理，需要进行安全把控下，建议运维人员针对每个项目进行不一样的盐值操作，避免一个项目泄露了，造成其他关联项的信息泄露。安全无大小呀，还是谨慎为妙！

## 参考资料

1. https://github.com/ulisesbocchio/jasypt-spring-boot
2. http://www.jasypt.org/
