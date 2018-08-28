---
layout: post
tags : [java, sprint]
title: Spring Boot 学习笔记

---

## 1. Spring

### 1.1 Bean

#### bean 的命名规则

Spring IOC 容器管理的每个 bean 都有一个唯一的标识符，该标识符既可以由 id 属性来指定，也可以由 name 属性指定，还可以由 Spring IOC 容器采用一定规则生成，具体规则如下:

* 如果 bean 的 id 和 name 属性都没有指定，那么 bean 的唯一标识符由 Spring IOC 容器自动生成
* 如果只定义了 bean 的 id 属性，那么该 id 就为 bean 的唯一标识符
* 如果只定义了 bean 的 name 属性，那么 name 属性的第一个值就为 bean 的唯一标识符
* 如果同时定义了 id 和 name 属性，那么 id 为 bean 的唯一标识符
* name 属性或者 alias 属性可以增加别名

#### Bean 的 Scope

* Singleton: 容器中只有一个Bean实例, 默认配置
* Prototype: 每次调用新建一个Bean
* Request: 每个http request新建一个Bean
* Session
* GlobalSession

### 1.2 Spring 注解

#### 声明Bean的注解:

* `@Component` 组件, 没有明确的角色 @Target(value=TYPE)

以下三者都组合了Component:

* `@Service` 业务逻辑层(service 层)
* `@Repository` 数据访问层(dao)
* `@Controller` 展示层(mvc)

* `@Bean` @Target(value={METHOD,ANNOTATION_TYPE})

  生成的bean的name默认是该方法名称

[@Component 和 @Bean 区别](http://stackoverflow.com/questions/10604298/spring-component-versus-bean)

#### 注入Bean注解:

以下3个注解通用

* `@Autowired` @Target(value={CONSTRUCTOR,METHOD,PARAMETER,FIELD,ANNOTATION_TYPE})
* `@Inject`
* `@resource`

### 1.3 配置类

* `@Configuration` 声明配置类
* `@ComponentScan(...)` 自动扫描指定包名下的声明Bean, 默认扫描配置类所在包

### 1.4 context

#### AnnotationConfigApplicationContext

TODO

#### 获得Bean

context.getBean(SomeClass.class);

context.getBean("BeanName");

---

## 2. Spring MVC

### 2.1 MVC 和 三层架构

MVC:

* M: 数据模型, 是偶包含数据的对象. 有个专门的类叫做Model, 用来和V交互
* V: 视图
* C: 控制器, 有@Controller注解

三层架构是由Spring框架负责管理的:

* 展示层: MVC
* 应用层: Service层
* 数据访问层: DAO层

### 2.2 注解

* `@Controller` : 类注解, 表明这个类是Spring MVC的Controller

  会将web请求映射到@RequestMapping的方法上

  在声明普通Bean时, @Component @Service @Repository @Controller是等同的, 不过MVC的控制器只能使用@Controller

* `@RequestMapping`: 类或方法注解, 映射web请求(路径和参数)

  方法注解的路径会集成类上注解的路径, 类注解是方法注解的前缀, 而不是覆盖

  单路由: `@RequestMapping("/someurl")` 多路由: `@RequestMapping({"url1", "url2"})`

  属性produces: 定制返回的response的每题类型和字符集, 如`produces="application/json;charset=UTF-8"`

  默认属性value中可以声明path variable: `value="/pathvar/{username}"`, 同时要求在action的形参上定义`@PathVariable`声明的path参数

* `@ResponseBody` 类或方法注解, 将返回值放在response body中, 而不是返回一个页面, 返回类型可能是json/xml

  此注解可以放在方法返回类型前, 或者方法上, 以及类上.

* `@RequestBody` 方法参数注解, 允许参数在request体内, 而不是在url后面

* `@PathVariable` 方法参数注解, 用于接收路径参数

* `@RestController` 组合@Controller和@ResponseBody

---

## 3 Spring Boot

### 3.1 特点

* 可以以jar包独立运行
* 内嵌servlet容器
* 提供了很多starter简化Maven配置
* 自动配置Spring: 根据classpath中的jar包和类, 为jar包中的类自动配置Bean
* 准生产环境的应用监控
* 无代码生产和xml配置


### 3.2 Spring boot 注解

* SpringBootApplication

  ```
  @Trget({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Inherited
  @SpringBootConfiguration
  @EnableAutoConfiguration: 根据类路径中的jar包依赖为当前项目进行自动配置
  @ComponentScan
  ```

* Value

  设置属性的值?

  获取配置文件中的配置项

  ```
  @Value("${server.port}")
  private String port;
  ```

* ConfigurationProperties

  TODO

---

## 配置

### 配置文件

* properties格式: src/main/resources/application.properties
* yaml格式: src/main/resources/application.yaml

常用配置:

* server.port 端口
* server.context-path: 根路径

### 命令行配置

在执行jar包时可以通过命令行参数改变配置:

`java -jar xx.jar --server.port=9090`

### Profile 配置

在`application.properties` 中指定Profile: `spring.profiles.active=prod`

同时指定的Profile文件必须存在: `application-{profile}.properties`

---

## Action 定义

方法形参列表可以包括:

* 普通url参数(?后面的参数)

* @PathVariable 定义的path参数

* 参数对象

* request对象: `HttpServletRequest request`

* response对象: `HttpServletResponse response`

各参数顺序任意

---

## Mybatis

pom.xml:

```xml
  <dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.1.1</version>
  </dependency>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.21</version>
  </dependency>
```

application.properties:

```xml
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

相关注解:

org.apache.ibatis.annotations

* MapperScan: 扫描mapper接口所在的包
* Mapper: 定义Mapper接口
* Select/Insert: mapper接口的方法注解, 用于声明sql
* Param: mapper接口的方法的参数注解, 用于对应sql中的参数

---

## Idea 快捷键

* `alt + cmd + L` 格式化代码
* `Ctrl + 回车` 呼出生成器
* `allt + 回车` 显示意向动作和快速修复代码
