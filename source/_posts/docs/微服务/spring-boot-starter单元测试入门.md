---
title: spring-boot-starter单元测试入门
date: 2019-05-06 19:33:36
categories: Java
tags:
- spring-boot
- spring-boot-starter-test
---

这次来介绍下Spring Boot中对单元测试的整合使用，本篇会通过以下3点来介绍，基本满足日常需求：

- Dao层单元测试
- Service层单元测试
- Controller层单元测试
在单元测试中要尽量使用断言，本文所有的测试类都符合几个原则：
- 测试类卸载src/test/java目录下
- 测试类的包结构与被测试类的包结构相同
- 测试类的命名都是被测试类类名后缀加上Test，例如,UserDaoImpl与UserDaoImplTest相对应
- 测试类的方法与被测试类的方法命名相同

<!-- more -->

## 1.前言

这次来介绍下Spring Boot中对单元测试的整合使用，本篇会通过以下3点来介绍，基本满足日常需求：

- Dao层单元测试
- Service层单元测试
- Controller层单元测试
在单元测试中要尽量使用断言，本文所有的测试类都符合几个原则：
- 测试类卸载src/test/java目录下
- 测试类的包结构与被测试类的包结构相同
- 测试类的命名都是被测试类类名后缀加上Test，例如,UserDaoImpl与UserDaoImplTest相对应
- 测试类的方法与被测试类的方法命名相同

## 2.正文

### 2.1核心依赖

在Spring Boot 项目中引入单元测试很简单，依赖如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 2.2 如何创建单元测试类

Spring Boot中单元测试类写在在src/test/java目录下，你可以手动创建具体测试类，如果是IDEA，则可以通过IDEA自动创建测试类，如下图，可以通过快捷键Ctrl+Shift+T(Window)来创建，如下：

### 2.3. 盲点解释

#### @Runwith

JUnit用例都是在Runner（运行器）来执行的。通过它，可以为这个测试类指定一个特定的Runner。
JUnit允许用户指定其它的单元测试执行类，只需要我们的测试执行类继承类org.junit.runners.BlockJUnit4ClassRunner就可以了，Spring的执行类SpringJUnit4ClassRunner就是继承了该类。我们平时用Spring也比较多，为了能够更加方便的引用配置文件，我们单元测试就使用了Spring实现的执行类。此时的单元测试执行类将会看起来是这样：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserDaoImplTest {

    @Autowired
    private UserDao dao;

    /**
     * 测试根据用户名查询用户
     */
    @Test
    public void getByUsername() {
        User user = dao.getByUsername("小明");
        System.err.println(user);
    }
```

#### SpringRunner

`SpringRunner`是`SpringJUnit4ClassRunner`的一个别名。`SpringJUnit4ClassRunner`是JUnit的`BlockJUnit4ClassRunner`类的一个常规扩展，提供了一些spring测试环境上下文去规范JUnit测试。

#### @SpringBootTest

注解制定了一个测试类运行了Spring Boot环境。提供以下特性：

- 自动搜索到SpringBootConfiguration注解的文件。
- 允许自动注入Environment类读取配置文件。
- 提供一个webEnvironment环境，可以完整的允许一个web环境使用随机的端口或者自定义的端口。

#### Spring Boot测试步骤

直接在测试类上面加上如下2个注解
`@RunWith(SpringRunner.class)`
`@SpringBootTest`
就能取到spring中的容器的实例，如果配置了@Autowired那么就自动将对象注入。

#### 单元测试回滚

单元个测试的时候如果不想造成垃圾数据，可以开启事物功能，记在方法或者类头部添加@Transactional注解即可,如下：

```java
@Transactional
@Test
public void save() {
    User user = new User("测试用户", "123456");
    int save = dao.save(user);
    Assert.assertEquals(1, save);
}
```

这样测试完数据就会回滚了，不会造成垃圾数据。

## 3.核心代码示例

### 3.1.Dao层单元测试

基本上所有的WEB程序都会涉及到数据库，本次示例就以最简化的模式：
持久层框架就用spring-jdbc。
Dao层的测试涉及到基本的CRUD操作。

#### UserDaoImpl

```java
@Repository
public class UserDaoImpl implements UserDao {

    @Autowired
    private JdbcTemplate template;

    private static final BeanPropertyRowMapper<User> MAPPER = new BeanPropertyRowMapper<>(User.class);

    /**
     * 新建用户
     *
     * @param user
     * @return
     */
    @Override
    public int save(User user) {
        return template.update(
                "insert into user(username, password) values (?, ?)",
                user.getUsername(), user.getPassword());
    }

    /**
     * 修改密码
     *
     * @return
     */
    @Override
    public int update(User user) {
        return template.update(
                "update user set password = ? where username = ?", user.getPassword(), user.getUsername());
    }

    /**
     * 根据ID删除用户
     *
     * @return
     */
    @Override
    public int delete(Long id) {
        return template.update("delete from user where id = ?", id);
    }

    /**
     * 获取所有用户
     *
     * @return
     */
    @Override
    public List<User> list() {
        return template.query("select id, username, password from user", MAPPER);
    }

    @Override
    public User getByUsername(String username) {
        try {
            return template.queryForObject(
                    "select id, username, password from user where username = ? limit 1", MAPPER, username);
        } catch (EmptyResultDataAccessException e) {
            return null;
        }
    }
}
```

#### UserDaoImplTest

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserDaoImplTest {

    @Autowired
    private UserDao dao;

    /**
     * 测试根据用户名查询用户
     */
    @Test
    public void getByUsername() {
        User user = dao.getByUsername("小明");
        System.err.println(user);
    }

    /**
     * 测试新建用户
     */
    @Transactional
    @Test
    public void save() {
        User user = new User("测试用户", "123456");
        int save = dao.save(user);
        Assert.assertEquals(1, save);
    }

    /**
     * 测试修改密码
     */
    @Transactional
    @Test
    public void update() {
        User user = new User("测试用户", "123456");
        dao.save(user);
        /* 密码修改前 */
        User before = dao.getByUsername("测试用户");
        user.setPassword("654321");
        /* 密码修改后 */
        dao.update(user);
        User after = dao.getByUsername("测试用户");
        /* 断言判断修改密码前后的两个类不同 */
        Assert.assertNotEquals(before, after);
    }

    /**
     * 测试删除用户
     */
    @Transactional
    @Test
    public void delete() {
        int save = dao.save(new User("测试用户", "123456"));
        Assert.assertEquals(1, save);
        User user = dao.getByUsername("测试用户");
        int delete = dao.delete(user.getId());
        Assert.assertEquals(1, delete);
    }

    /**
     * 测试用户列表
     */
    @Test
    public void list() {
        List<User> list = dao.list();
        System.err.println(list);
    }
}
```

### 3.2.Service层单元测试

#### UserServiceImpl

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao dao;

    /**
     * 根据用户名查询用户
     *
     * @param username
     * @return
     */
    @Override
    public User getByUsername(String username) {
        return dao.getByUsername(username);
    }

    /**
     * 列表查询用户
     *
     * @return
     */
    @Override
    public List<User> list() {
        return dao.list();
    }

    /**
     * 修改密码
     * @return
     */
    @Transactional
    @Override
    public int updatePassword(User user) {
        return dao.update(user);
    }

    /**
     * 根据ID删除用户
     * @return
     */
    @Transactional
    @Override
    public int deleteById(Long id) {
        return dao.delete(id);
    }

    /**
     * 添加用户
     * @param user
     * @return
     */
    @Transactional
    @Override
    public int save(User user) {
        return dao.save(user);
    }
}
```

#### UserServiceImplTest

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserServiceImplTest {

    @Autowired
    private UserService service;

    /**
     * 测试根据用户名查询用户
     */
    @Test
    public void getByUsername() {
        User user = service.getByUsername("小明");
        System.err.println(user);
    }

    /**
     * 测试列表查询用户
     */
    @Test
    public void list() {
        List<User> list = service.list();
        System.err.println(list);
    }

    /**
     * 测试修改密码
     */
    @Transactional
    @Test
    public void updatePassword() {
        User user = new User("测试用户", "123456");
        service.save(user);
        /* 密码修改前 */
        User before = service.getByUsername("测试用户");
        user.setPassword("654321");
        /* 密码修改后 */
        service.updatePassword(user);
        User after = service.getByUsername("测试用户");
        /* 断言判断修改密码前后的两个类不同 */
        Assert.assertNotEquals(before, after);
    }

    /**
     * 测试根据ID删除用户
     */
    @Transactional
    @Test
    public void deleteById() {
        int save = service.save(new User("测试用户", "123456"));
        Assert.assertEquals(1, save);
        User user = service.getByUsername("测试用户");
        int delete = service.deleteById(user.getId());
        Assert.assertEquals(1, delete);
    }

    /**
     * 测试添加用户
     */
    @Transactional
    @Test
    public void save() {
        List<User> list = service.list();
        System.err.println(list);
    }
}
```

### 3.3.Controller层单元测试

上面只是针对Service和Dao层做测试，但是有时候需要对Controller层（API）做测试，这时候就得用到MockMvc了，你可以不必启动工程就能测试这些接口。

MockMvc实现了对Http请求的模拟，能够直接使用网络的形式，转换到Controller的调用，这样可以使得测试速度快、不依赖网络环境，而且提供了一套验证的工具，这样可以使得请求的验证统一而且很方便。

#### UserController

```java
@RestController
public class UserController {

    @Autowired
    private UserService service;

    /**
     * 用户列表
     * @return
     */
    @RequestMapping("/user/list")
    public List<User> list() {
        return service.list();
    }

    /**
     * 新建用户
     * @param username
     * @param password
     * @return
     */
    @RequestMapping("/user/save")
    public int save(@RequestParam String username, @RequestParam String password) {
        return service.save(new User(username, password));
    }

    /**
     * 根据ID删除用户
     * @param id
     * @return
     */
    @RequestMapping("/user/delete")
    public int delete(@RequestParam Long id) {
        return service.deleteById(id);
    }

    /**
     * 根据用户名查询用户
     * @param username
     * @return
     */
    @RequestMapping("/user/getByUsername")
    public User getByUsername(@RequestParam String username) {
        return service.getByUsername(username);
    }

    /**
     * 根据用户名修改密码
     * @param username
     * @param newPassword
     * @return
     */
    @RequestMapping("/user/updatepassword")
    public int updatepassword(@RequestParam String username, @RequestParam String newPassword) {
        return service.updatePassword(new User(username, newPassword));
    }

}
```

#### UserControllerTest

```java
/**
 * 1.SpringRunner继承SpringJUnit4ClassRunner，通俗的讲SpringRunner是SpringJUnit4ClassRunner的别名。
 * 2.SpringBoot项目中自动添加注解式事务支持，在测试环境中添加@Transactional，可以对有事务影响的数据库操作回滚
 */
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class UserControllerTest {

    /**
     * 模拟浏览器
     */
    MockMvc mvc;

    /**
     * 加载WEB的上下文
     */
    @Autowired
    WebApplicationContext context;

    @Before
    public void before() {
        //       mvc = MockMvcBuilders.standaloneSetup(new TestController()).build();
        mvc = MockMvcBuilders.webAppContextSetup(context).build();//建议使用这种
    }

    /**
     * 测试用户列表
     *
     * @throws Exception
     */
    @Test
    public void list() throws Exception {
        mvc.perform(
                MockMvcRequestBuilders.get("/user/list"))
                .andExpect(status().isOk()) // 期待返回状态码200
                .andDo(print()); // 打印返回的 http response 信息
    }

    /**
     * 测试添加用户
     * 期待：Body = 1
     *
     * @throws Exception
     */
    @Transactional
    @Test
    public void save() throws Exception {
        mvc.perform(
                MockMvcRequestBuilders.post("/user/save")
                        .param("username", "测试用户1")
                        .param("password", "123456"))
                .andExpect(status().isOk())
                .andDo(print());
    }

    /**
     * 测试删除用户
     * 期待：Body = 1
     */
    @Transactional
    @Test
    public void delete() throws Exception {
        mvc.perform(MockMvcRequestBuilders.post("/user/delete")
                .param("id", "1"))
                .andExpect(status().isOk())
                .andDo(print());
    }

    /**
     * 测试根据用户名查询用户
     */
    @Test
    public void getByUsername() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/user/getByUsername")
                .param("username", "小明"))
                .andExpect(status().isOk())
                .andDo(print());
    }

    /**
     * 测试修改密码
     * 期待：Body = 1
     */
    @Transactional
    @Test
    public void updatepassword() throws Exception {
        mvc.perform(MockMvcRequestBuilders.post("/user/updatepassword")
                .param("username", "小明")
                .param("newPassword", "654321"))
                .andExpect(status().isOk())
                .andDo(print());
    }
}
```