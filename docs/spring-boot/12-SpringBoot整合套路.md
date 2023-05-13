
## 前言
网上有很多文章都在说`Spring Boot 如何整合 xxx`，有文章教你为什么这么整合吗？整合了千万个框架，其实套路就那么几个，干嘛要学千万个，不如来这学习几个套路轻松整合，它不香吗？？？

今天写这篇文章的目的就是想从思想上教给大家几个套路，不用提到整合什么就去百度了，自己尝试去亲手整合一个。

## Spring Boot 版本
本文基于的Spring Boot的版本是`2.3.4.RELEASE`。

## 1. 找到自动配置类
Spring Boot 在整合任何一个组件的时候都会先添加一个依赖`starter`，比如整合的Mybatis有一个`mybatis-spring-boot-starter`，依赖如下：
```xml
<dependency>
         <groupId>org.mybatis.spring.boot</groupId>
         <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.0.0</version>
</dependency>
```

每一个`starter`基本都会有一个自动配置类，命名方式也是类似的，格式为：`xxxAutoConfiguration`，比如Mybatis的自动配置类就是`MybatisAutoConfiguration`，`Redis`的自动配置类是`RedisAutoConfiguration`，`WEB`模块的自动配置类是`WebMvcAutoConfiguration`。


## 2. 注意@Conditionalxxx注解
`@Conditionalxxx`标注在配置类上或者结合`@Bean`标注在方法上，究竟是什么意思，在上一篇文章[这类注解都不知道，还好意思说会Spring Boot](https://mp.weixin.qq.com/s/BoujdCIHPK79jT9RKAmyug)已经从表层到底层深入的讲了一遍，不理解的可以查阅一下。

> 首先需要注意自动配置类上的`@Conditionalxxx`注解，这个是自动配置类生效的条件。

比如`WebMvcAutoConfiguration`类上标了一个如下注解：
```java
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
```

以上这行代码的意思就是当前IOC容器中没有`WebMvcConfigurationSupport`这个类的实例时自动配置类才会生效，这也就是在配置类上标注`@EnableWebMvc`会导致自动配置类`WebMvcAutoConfiguration`失效的原因。

> 其次需要注意方法上的`@Conditionalxxx`注解，Spring Boot会在自动配置类中结合`@Bean`和`@Conditionalxxx`注解提供一些组件运行的默认配置，但是利用`@Conditionalxxx`（在特定条件下生效）注解的`条件性`，方便开发者覆盖这些配置。

比如在Mybatis的自动配置类`MybatisAutoConfiguration`中有如下一个方法：
```java
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {}
```
以上这个方法不用看方法体的内容，只看方法上的注解。`@Bean`这个注解的意思是注入一个`Bean`到`IOC容器`中，`@ConditionalOnMissingBean`这个注解就是一个条件判断了，表示当`SqlSessionFactory`类型的对象在`IOC容器`中不存在才会注入。

哦？领悟到了吧，**言外之意就是如果开发者需要定制`SqlSessionFactory`，则可以自己的创建一个`SqlSessionFactory`类型的对象并且注入到IOC容器中即能覆盖自动配置类中的**。比如在Mybatis配置多数据源的时候就需要定制一个`SqlSessionFactory`而不是使用自动配置类中的。

> 总之，一定要注意自动配置类上或者方法上的`@Conditionalxxx`注解，这个注解表示某种特定条件。

下面列出了常用的几种注解，如下：

1. `@ConditionalOnBean`：当容器中有指定Bean的条件下进行实例化。
2. `@ConditionalOnMissingBean`：当容器里没有指定Bean的条件下进行实例化。
3. `@ConditionalOnClass`：当classpath类路径下有指定类的条件下进行实例化。
4. `@ConditionalOnMissingClass`：当类路径下没有指定类的条件下进行实例化。
5. `@ConditionalOnWebApplication`：当项目是一个Web项目时进行实例化。
6. `@ConditionalOnNotWebApplication`：当项目不是一个Web项目时进行实例化。
7. `@ConditionalOnProperty`：当指定的属性有指定的值时进行实例化。
8. `@ConditionalOnExpression`：基于SpEL表达式的条件判断。
9. `@ConditionalOnJava`：当JVM版本为指定的版本范围时触发实例化。
10. `@ConditionalOnResource`：当类路径下有指定的资源时触发实例化。
11. `@ConditionalOnJndi`：在JNDI存在的条件下触发实例化。
12. `@ConditionalOnSingleCandidate`：当指定的Bean在容器中只有一个，或者有多个但是指定了首选的Bean时触发实例化。

## 3. 注意EnableConfigurationProperties注解

`EnableConfigurationProperties`这个注解常标注在配置类上，使得`@ConfigurationProperties`标注的配置文件生效，这样就可以在全局配置文件（`application.xxx`）配置指定前缀的属性了。

在Redis的自动配置类`RedisAutoConfiguration`上方标注如下一行代码：
```java
@EnableConfigurationProperties(RedisProperties.class)
```
这行代码有意思了，我们可以看看`RedisProperties`的源码，如下：
```java
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {
	private int database = 0;
	private String url;
	private String host = "localhost";
	private String password;
  .....
```

`@ConfigurationProperties`这个注解指定了全局配置文件中以`spring.redis.xxx`为前缀的配置都会映射到`RedisProperties`的指定属性中，其实`RedisProperties`这个类中定义了Redis的一些所需属性，比如`host`，`IP地址`，`密码`等等。

**`@EnableConfigurationProperties`注解就是使得指定的配置生效，能够将全局配置文件中配置的属性映射到相关类的属性中。**

**为什么要注意`@EnableConfigurationProperties`这个注解呢？**

> 引入一个组件后往往需要改些配置，我们都知道在全局配置文件中可以修改，但是不知道前缀是什么，可以改哪些属性，因此找到`@EnableConfigurationProperties`这个注解后就能找到对应的配置前缀以及可以修改的属性了。


## 4. 注意@Import注解

这个注解有点牛逼了，`Spring 3.x`中就已经有的一个注解，大致的意思的就是快速导入一个Bean或者配置类到IOC容器中。这个注解有很多妙用，后续会单独写篇文章介绍下。

> `@Import`这个注解通常标注在自动配置类上方，并且一般都是导入一个或者多个配置类。

比如`RabbitMQ`的自动配置类`RabbitAutoConfiguration`上有如下一行代码：
```java
@Import(RabbitAnnotationDrivenConfiguration.class)
```
这行代码的作用就是添加了`RabbitAnnotationDrivenConfiguration`这个配置类，使得Spring Boot在加载到自动配置类的时候能够一起加载。

比如Redis的自动配置类`RedisAutoConfiguration`上有如下一行代码：
```java
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
```
这个`@Import`同时引入了`Lettuce`和`Jedis`两个配置类了，因此如果你的Redis需要使用Jedis作为连接池的话，想要知道Jedis都要配置什么，此时就应该看看`JedisConnectionConfiguration`这个配置类了。

> **总结**：`@Import`标注在自动配置类上方，一般都是快速导入一个或者多个配置类，因此如果自动配置类没有配置一些东西时，一定要看看`@Import`这个注解导入的配置类。

## 5. 注意@AutoConfigurexxx注解
`@AutoConfigurexxx`这类注解决定了自动配置类的加载顺序，比如`AutoConfigureAfter`（在指定自动配置类之后）、`AutoConfigureBefore`（在指定自动配置类之前）、`AutoConfigureOrder`（指定自动配置类的优先级）。

> 为什么要注意顺序呢？因为某些组件往往之间是相互依赖的，比如`Mybatis`和`DataSource`，肯定要先将数据源相关的东西配置成功才能配置`Mybatis`吧。`@AutoConfigurexxx`这类注解正是解决了组件之间相互依赖的问题。

比如`MybatisAutoConfiguration`上方标注了如下一行代码：
```java
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
```

这个行代码意思很简单，就是`MybatisAutoConfiguration`这个自动配置在`DataSourceAutoConfiguration`这个之后加载，因为你需要我，多么简单的理由。

好了，这下明白了吧，以后别犯傻问：**为什么Mybatis配置好了，启动会报错**？这个问题先看看数据源有没有配置成功吧。


## 6. 注意内部静态配置类
有些自动配置类比较简单没那么多套路，比如`RedisAutoConfiguration`这个自动配置类中就定义了两个注入Bean的方法，其他的没了。

但是有些自动配置类就没那么单纯了，中间能嵌套`n`个静态配置类，比如`WebMvcAutoConfiguration`，类中还嵌套了`WebMvcAutoConfigurationAdapter`、`EnableWebMvcConfiguration`、`ResourceChainCustomizerConfiguration`这三个配置类。如果你光看`WebMvcAutoConfiguration`这个自动配置类好像没配置什么，但是其内部却是大有乾坤啊。

> **总结**：一定要自动配置类的内部嵌套的配置类，真是大有乾坤啊。

## 总结

以上总结了六条整合的套路，希望能够帮助读者摆脱百度，自己也能独立整合组件。

总之，Spring Boot整合xxx组件的文章很多，相信大家也看的比较懵，其实套路都是一样，学会陈某分享的套路，让你少走弯路！！！




