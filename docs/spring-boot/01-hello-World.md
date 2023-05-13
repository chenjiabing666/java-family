

## 前言

相信从事Java开发的朋友都听说过`SSM`框架，这还算年轻的，老点的甚至经历过`SSH`，说起来有点恐怖，哈哈。比如我就是经历过`SSH`那个时代末流，没办法，很无奈。

当然无论是SSM还是SSH都不是今天的重点，今天要说的是`Spring Boot`，一个令人眼前一亮的框架，从大的说，Spring Boot取代了`SSM` 中的`SS`的角色。

今天这篇文章就来谈谈Spring Boot，这个我第一次使用直呼`爽`的框架。

## 什么是Spring Boot？

`Spring Boot` 是由 Pivotal 团队提供的全新框架。`Spring Boot` 是所有基于 `Spring Framework 5.0` 开发的项目的起点。`Spring Boot` 的设计是为了让你尽可能快的跑起来`Spring` 应用程序并且尽可能减少你的配置文件。

**Spring Boot 的设计目的简单一句话：简化Spring应用的初始搭建以及开发过程。**

从最根本上来讲，Spring Boot 就是一些库的集合，它能够被任意项目的构建系统所使用。它使用 “**约定大于配置**” （项目中存在大量的配置，此外还内置一个习惯性的配置）的理念让你的项目快速运行起来。

**约定大于配置**这个如何理解？其实简单的来说就是Spring Boot在搭建之初就内置了许多实际开发中的常用配置，只有少部分的配置需要开发人员自己去配置。

## 如何搭建一个Spring Boot项目？

其实搭建一个SpringBoot项目有很多种方式，最常见的两种方式如下：
    1. 创建Maven项目，自己引入依赖，创建启动类和配置文件。
        2. 直接IDEA中的` Spring Initializr`创建项目。

**第一种方式不适合入门的朋友玩，今天演示第二种方式搭建一个Spring Boot项目。**

第一步在IDEA中选择`File-->NEW-->Project`，选择`Spring Initializr`，指定`JDK`版本`1.8`，然后`Next`。如下图：
![](https://img.java-family.cn/第一弹/1.png)

第二步指定Maven坐标、包名、JDK版等信息，然后`Next`，如下图：
![](https://img.java-family.cn/第一弹/2.png)

第三步选择自己所需要的依赖、Spring Boot的版本，Spring Boot与各个框架适配都是以`starter`方式，这里我们选择WEB开发的所需的`starter`即可，如下图：
![](https://img.java-family.cn/第一弹/3.png)

第四步指定项目的名称，路径即可完成，点击`Finish`等待创建成功，如下图：
![](https://img.java-family.cn/第一弹/4)

创建成功的项目如下图：
![5](https://img.java-family.cn/第一弹/5)

其中的`DemoApplication`是项目的启动类，里面有一个`main()`方法就是用来启动Spring Boot。`application.properties`是Spring Boot的配置文件。

此时可以启动项目，在`DemoApplication`运行`main`方法即可启动，启动成功如下图：
![](https://img.java-family.cn/第一弹/6)

由于SpringBoot默认内置了Tomcat，因此启动的默认端口就是`8080`。

## 第一个程序 Hello World

学习任何一种技术总是要问候一下世界，哈哈..........

既然是WEB开发，就写个接口吧，前面创建的时候已经引用了`WEB`的`starter`，如果没有引用，则可以在`pom.xml`引入以下依赖：
```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

```

- 下面写一个`HelloWorldController`如下：
```java
package com.example.demo.controller;
@RestController
public class HelloWorldController {
    @RequestMapping("/hello")
    public String helloWorld(){
        return "Hello World";
    }
}
```

`@RestController`：标记这是一个`controller`，是`@Controller`和
`@ResponseBody`这两个注解的集合。

`@RequestMapping`：指定一个映射

**以上两个注解都是Spring中的，这里就不再细说了。**

由于内置的Tomcat默认端口是`8080`，所以启动项目，访问`http://127.0.0.1:8080/hello`即可。


## 依赖解读

Spring Boot项目中的`pom.xml`中有这么一个依赖，如下：
```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
        <relativePath/>
    </parent>
```

`<parent>`这个标签都知道什么意思，`父亲`是吧，这么个标签主要的作用就是用于版本控制。这也就是引入的`WEB`模块`starter`的时候不用指定版本号`<version>`标签的原因，因为在`spring-boot-starter-parent`中已经指定了，类似于一种继承的关系，父亲已经为你提供了，你只需要选择用不用就行。

**为什么引入`spring-boot-starter-web`就能使用`Spring mvc`的功能呢？**

这确实是个难以理解的问题，为了理解这个问题，我们不妨看一下`spring-boot-starter-web`这个启动器都依赖了什么？如下：
```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.3.4.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-json</artifactId>
      <version>2.3.4.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <version>2.3.4.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>5.2.9.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.2.9.RELEASE</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
```

看到这应该明白了吧，`spring-boot-starter-web`这个`starter`中其实内部引入了`Spring`、`springmvc`、`tomcat`的相关依赖，当然能够直接使用Spring MVC相关的功能了。


## 什么是配置文件？
前面说过`application.properties`是Spring Boot的配置文件，那么这个配置文件究竟是配置什么的呢？

其实Spring Boot为了能够适配每一个组件，都会提供一个`starter`，但是这些启动器的一些信息不能在内部写死啊，比如数据库的用户名、密码等，肯定要由开发人员指定啊，于是就统一写在了一个`Properties`类中，在Spring Boot启动的时候根据`前缀名+属性名称`从配置文件中读取，比如`WebMvcProperties`，其中定义了一些Spring Mvc相关的配置，前缀是`spring.mvc`。如下：
```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
```

那么我们需要修改Spring Mvc相关的配置，只需要在`application.properties`文件中指定`spring.mvc.xxxx=xxxx`即可。

**其实配置文件这块还是有许多道道儿的，后面文章会详细介绍。**

## 什么是启动类？
前面说过启动类是`DemoApplication`，源码如下：
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@SpringBootApplication`是什么？其实一眼看上去，这个类在平常不过了，唯一显眼的就是`@SpringBootApplication`这个注解了，当然主要的作用还真是它。这个注解的源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```

我滴乖乖儿，注解叠加啊，完全是由`@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan`这三个注解叠加而来。

**`ComponentScan`**：这个注解并不陌生，Spring中的注解，包扫描的注解，这个注解的作用就是在项目启动的时候扫描**启动类的同类级以及下级包中的Bean**。

**`@SpringBootConfiguration`**：这个注解使Spring Boot的注解，源码如下：
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

从源码可以看出，`@SpringBootConfiguration`完全就是的`@Configuration`注解，`@Configuration`是Spring中的注解，表示该类是一个配置类，因此我们可以在启动类中做一些配置类可以做的事，比如注入一个`Bean`。

**`@EnableAutoConfiguration`**：这个注解看到这个名字就知道怎么回事了，直接翻译码，**开启自动配置**，真如其名，源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

又是一个熟悉的注解`@Import`，什么功能呢？**快速导入Bean到IOC容器中**，有三种方式，这里用的是其中一种`ImportSelector`方式。不是本文重点，不再细说。

`@EnableAutoConfiguration`这个注解的作用也就一目了然了，无非就是`@Import`的一种形式而已，在项目启动的时候向IOC容器中快速注入`Bean`而已。

**好了，启动类就先介绍到这，后续讲到源码文章才能更清楚的了解到这个类的强大之处。**


## 如何进行单元测试？

Spring Boot项目创建之处为我们提供了一个单元测试的类，如下：
```java
@SpringBootTest
class DemoApplicationTests {

    @Test
    void contextLoads() {
    }

}
```

`@SpringBootTest`：这个注解指定这个类是单元测试的类。

在这个类中能够自动的获取IOC容器中的Bean，比如：
```java
@SpringBootTest
class DemoApplicationTests {

    @Autowired
    private HelloWorldController helloWorldController;
```

简单的介绍下而已，实际开发中用不到，随着项目越来越大，启动的时间越来越长，谁会傻到启动一个测试方法来检验代码，纯粹浪费时间。



## 总结
作为Spring Boot的第一弹，写到这儿就结束了，没什么的深入的内容，只是简单的对Spring Boot做了初步的了解。
