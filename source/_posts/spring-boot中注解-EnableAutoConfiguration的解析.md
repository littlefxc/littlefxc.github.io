---
title: spring-boot中注解@EnableAutoConfiguration的解析
date: 2019-05-06 19:49:41
categories: Java
tags:
- spring-boot
---

个人感觉@EnableAutoConfiguration这个Annotation最为重要，Spring框架有提供的各种名字为@Enable开头的Annotation定义，比如@EnableScheduling、@EnableCaching、@EnableMBeanExport等，@EnableAutoConfiguration的理念和做事方式其实一脉相承，简单概括一下就是，借助@Import的支持...

<!-- more -->

个人感觉@EnableAutoConfiguration这个Annotation最为重要，Spring框架有提供的各种名字为@Enable开头的Annotation定义，比如@EnableScheduling、@EnableCaching、@EnableMBeanExport等，@EnableAutoConfiguration的理念和做事方式其实一脉相承，简单概括一下就是，**借助@Import的支持，收集和注册特定场景相关的bean定义**。

又例如：
- @EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器。
- @EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器。

而@EnableAutoConfiguration也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器，仅此而已！

@EnableAutoConfiguration作为一个复合Annotation,其自身定义关键信息如下：
![@EnableAutoCOnfiguration](https://img-blog.csdnimg.cn/20181217104614152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

其中，最关键的要属@Import(EnableAutoConfigurationImportSelector.class)，借助EnableAutoConfigurationImportSelector，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。

借助于Spring框架原有的一个工具类：SpringFactoriesLoader的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181217104843615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

**自动配置幕后英雄：SpringFactoriesLoader详解**

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从指定的配置文件META-INF/spring.factories加载配置。
```java
public abstract class SpringFactoriesLoader {  
    //...  
    public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {  
    ...  
    }  
  
  
    public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {  
    ....  
    }  
}  
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key,获取对应的一组@Configuration类

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181217105056281.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

上图就是从SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件中摘录的一段内容，可以很好地说明问题。
所以，@EnableAutoConfiguration自动配置就是：从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

**注：@Conditional派生注解（Spring注解版原生的@Conditional作用）**

作用：必须是@Conditional指定的条件成立，才给容器中添加组件，配置配里面的所有内容才生效；

| 注解  | 作用 |
| --- | --- |
|@Conditional扩展注解	 |作用（判断是否满足当前指定条件）|
|@ConditionalOnJava	|系统的java版本是否符合要求|
|@ConditionalOnBean	|容器中存在指定Bean|
|@ConditionalOnMissingBean	|容器中不存在指定Bean|
|@ConditionalOnExpression	|满足SpEL表达式指定|
|@ConditionalOnClass	|系统中有指定的类|
|@ConditionalOnMissingClass	|系统中没有指定的类|
|@ConditionalOnSingleCandidate	|容器中只有一个指定的Bean，或者这个Bean是首选Bean|
|@ConditionalOnProperty	|系统中指定的属性是否有指定的值|
|@ConditionalOnResource	|类路径下是否存在指定资源文件|
|@ConditionalOnWebApplication	|当前是web环境|
|@ConditionalOnNotWebApplication	|当前不是web环境|
|@ConditionalOnJndi	|JNDI存在指定项|
