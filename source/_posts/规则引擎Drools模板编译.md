---
title: 规则引擎Drools模板编译
date: 2019-05-05 20:14:08
categories: Java
tags:
- 规则引擎
---

## 规则引擎 Drools 模板编译

## 1.模板编译

### 1.1依赖

```xml
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-templates</artifactId>
    <version>7.7.0.Final</version>
</dependency>
```

### 1.2建立模板

新建模板文件 test.drt

```java
template header
condition
execution

package template

template "this is a test"

    rule "test_@{row.rowNumber}"
        when
            @{condition}
        then
            System.out.println(@{execution});
    end

end template
```

说明
规则模板主要由两部分构成：

Template header 定义了在模板中使用的变量。
模板中以 “template name” 开头, 以”end template” 结尾, 中间定义了模板的内容。变量占位符使用 @{variable_name} .
@{row.rowNumber}是一个特殊的变量, 每次会按顺序生成一个行号, 可用于区分规则名。

## 1.3 渲染模板生成规则文件

渲染模板的流程，先将数据封装为 DataProvider，然后通过 DataProviderCompiler 使用 DataProvider 将模板编译为 DRL。

Drools支持数组类型的DataProvider, ArrayDataProvider实现了DataProvider,  示例

 ```java
InputStream templateStream = DataDrivenTemplateExample.class.getResourceAsStream("/rules/SimpleTemplateExample.drt");
// @{row.rowNumber}=数组下标, @{condition}=规则条件, @{execution}=规则动作
DataProvider data = new ArrayDataProvider(new String[][]{
    new String[]{"String(this == \"规则条件\")", "\"规则动作\""}
});
DataProviderCompiler converter = new DataProviderCompiler();
String drl = converter.compile(data, templateStream);
 ```

### 1.4 编译规则

在模板渲染、编译成规则文件后，就可以正常的编译DRL规则文件， 新建会话等。
 KieHelper 是 Drools提供的工具类, 可用于编译DRL规则文件， 新建会话等。
ps: 也可以使用其他的方式编译，这里只是为了简单

```java
KieHelper helper = new KieHelper();
helper.addContent(drl, ResourceType.DRL);
KieSession kieSession = helper.build().newKieSession();
kieSession.insert(new String("Hello, World!"));
kieSession.fireAllRules();
kieSession.dispose();
```

### 1.5 模板编译示例

#### 1.5.1 DRT模板文件 template.drt

```java
template header
condition
execution

package template

template "this is a test"

    rule "test_@{row.rowNumber}"
        when
            @{condition}
        then
            System.out.println(@{execution});
    end

end template
```

#### 1.5.2 单元测试

```java
public class Demo {
    static KnowledgeBuilder builder;
KieBase kieBase;

    @BeforeClass
    public static void beforeClass() {
        builder = KnowledgeBuilderFactory.newKnowledgeBuilder();
    }

    @Before
    public void setUp() {
        /**
         * 1. 从本地模板DRT文件得到输入流
         * 2. 创建一个 ArrayDataProvider , 二维数组中元素按顺序与DRT文件中定义的变量一一对应
         * 3. 创建一个 DataProviderCompiler 对象用compile()方法渲染, 将二维数组中的元素一一填充到DRT模板中, 得到DRL(规则)字符串
         * 4. 加载DRL(规则)字符串
         */
        String pathToDrt = "E:\\template.drt";
        try (InputStream stream = new FileInputStream(pathToDrt)) {
            DataProvider dataProvider = new ArrayDataProvider(new String[][]{
                    new String[]{"String(this == \"规则条件\")", "\"规则动作\""}
            });
            DataProviderCompiler compiler = new DataProviderCompiler();
            String DRL = compiler.compile(dataProvider, stream);
            System.out.println("-----模板DRT渲染后的DRL-----");
            System.out.println(DRL);
            System.out.println("-----模板DRT渲染后的DRL-----");
            byte[] drlBytes = DRL.getBytes("UTF-8");
            Resource resourceTemplate = ResourceFactory.newByteArrayResource(drlBytes);
            builder.add(resourceTemplate, ResourceType.DRL);
        } catch (IOException e) {
            e.printStackTrace();
        }
        builder.add(resourceNative, ResourceType.DRL);
//        builder.add(resourceRemote, ResourceType.DRL);
        kieBase = builder.newKieBase();
    }

    @Test
    public void testTemplate() {
        KieSession kieSession = kieBase.newKieSession();
        kieSession.insert("规则条件");
        kieSession.fireAllRules();
        kieSession.dispose();
    }
}
```

#### 1.5.3 输出

>
    -----模板DRT渲染后的DRL-----
    package template
    rule "test_0"
        when
            String(this == "规则条件")
        then
            System.out.println("规则动作");
    end
    -----模板DRT渲染后的DRL-----
    规则动作

## 2 本地加载与远程加载

### 2.1 创建本地DRL文件

创建远程的DRL文件，地址为 E:\\native.drl

```java
package rules.native;
dialect  "mvel"

rule "native"
    when
    then
        System.out.println("本地加载成功");
end
```

### 2.2 创建远程DRL文件

创建远程的DRL文件，地址为 [http://localhost:8761/remote.drl](http://localhost:8761/remote.drl)

```java
package rules.remote;
dialect  "mvel"

rule "remote"
    when
    then
        System.out.println("远程加载成功");
end
```

### 2.3 单元测试

```java
public class Demo {

    static KnowledgeBuilder builder;
    KieBase kieBase;

    @BeforeClass
    public static void beforeClass() {
        builder = KnowledgeBuilderFactory.newKnowledgeBuilder();
    }

    @Before
    public void setUp() {
        /**
         * 本地加载DRL
         */
        Resource resourceNative = ResourceFactory.newFileResource("E:\\native.drl");
        /**
         * 远程加载URL
         */
        String url = "http://localhost:8761/remote.drl";
Resource resourceRemote = ResourceFactory.newUrlResource(url );
// 加载方式的不同
builder.add(resourceNative, ResourceType.DRL);
builder.add(resourceRemote, ResourceType.DRL);
    }

    @Test
    public void testNativeAndRemote() {
        KieSession kieSession = kieBase.newKieSession();
        kieSession.fireAllRules();
        kieSession.dispose();
    }}
```

### 2.4 输出

>远程加载成功
>本地加载成功

## 3 总结

关键API：
Resource: 资源类，规则文件的加载 
KnowledgeBuilder: 收集编译已经编写好的规则文件(drl)

从整体的收集、编译、执行上看，远程加载与本地加载大同小异，无非就是在使用Resource时加载规则文件上的不同。使用模板则在此基础上，需要将模板(drt)编译成规则(drl)。