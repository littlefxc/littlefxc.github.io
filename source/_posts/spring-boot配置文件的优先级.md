---
title: spring-boot配置文件的优先级
date: 2019-05-07 19:27:30
categories: Java
tags:
- spring-boot
---

# SpringBoot配置文件的优先级

## 项目结构

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321160947190.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

## 配置文件的优先级

application.properties 和 application.yml 文件按照优先级从大到小顺序排列在以下四个位置：

1. file:./config/ (当前项目路径config目录下);
2. file:./ (当前项目路径下);
3. classpath:/config/ (类路径config目录下);
4. classpath:/ (类路径config下).

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321160812255.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

源代码展示：

```java
public class ConfigFileApplicationListener
		implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {
// Note the order is from least to most specific (last one wins)
private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/";

// 省略其它代码
}
```

以端口配置为例

- 在resources/config目录下配置文件设置端口为8888;
- 在resources/目录下配置文件设置端口为8080;
- 在类路径config目录下配置文件设置端口为6666;
- 在类路径下配置文件设置端口为5555;

运行结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321160853993.png)


## 自定义配置文件的绑定

1. CustomizedFile 类

    ```java
    /**
     * 自定义配置文件, 需要配合使用后@Configuration和@PropertySource("classpath:customized-file.properties")来指定
     * @author fengxuechao
     */
    @Configuration
    @ConfigurationProperties(prefix = "customizedFile")
    @PropertySource("classpath:customized-file-${spring.profiles.active}.properties")
    public class CustomizedFile {
        private String name;
        private String author;
        private String path;
        private String description;
        // 省略 setter/getter
    }
    ```
    
    看到 `${spring.profiles.active}`，聪明的你一定知道这是 spring boot多环境自定义配置文件的实现方式。
    生效的配置文件是 `${spring.profiles.active}` 所指定的配置文件，本文案例中生效的是 `customized-file-dev.properties`。
    接下来继续创建配置文件验证

2. customized-file.properties

    ```yaml
    customizedFile.name=自定义配置文件名
    customizedFile.author=作者名
    customizedFile.path=路径地址
    customizedFile.description=看到这个就表明自定义配置文件成功了
    ```

3. customized-file-dev.properties

    ```yaml
    customizedFile.description=DEV:看到这个就表明自定义配置文件成功了
    ```

4. 运行结果：

	![在这里插入图片描述](https://img-blog.csdnimg.cn/2019032116083452.png)
	
	结论：只有 `customized-file-dev.properties` 中配置的属性生效
