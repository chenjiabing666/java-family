## 前言
今天是Spring Boot专栏的第五篇文章，相信大家看了前四篇文章对Spring Boot已经有了初步的了解，今天这篇文章就来介绍一下Spring Boot的重要功能WEB开发。

## Spring Boot 版本
本文基于的Spring Boot的版本是`2.3.4.RELEASE`。

## 前提条件（必须注意）
Spring Boot的WEB开发有自己的启动器和自动配置，最好采用Spring Boot的一套配置，这里千万不要在任何一个配置类上添加`@EnableWebMvc`这个注解，具体原因会单独一篇文章讲述。

**此篇文章所有的内容都是在没有标注`@EnableWebMvc`这个注解的前提下。**


## 添加依赖
Spring Boot对web模块有一个启动器，只需要在pom.xml中引入即可，如下：
```java
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**这个依赖看似只是引入了一个依赖，其实内部引入了Spring,Spring MVC的相关依赖，Spring Boot的启动器就是这么神奇，后面的文章会介绍启动器的原理和如何自定义启动器。**

## 第一个接口开发
假设这么一个需求，需要根据用户的ID获取用户信息，我们应该如何写接口呢？

其实和Spring MVC开发步骤一样，写一个controller，各种注解骚操作搞起，如下：
```java
@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/{id}")
    public Object getById(@PathVariable("id") String id){
        return User.builder()
                .id(id)
                .name("不才陈某")
                .age(18)
                .birthday(new Date())
                .build();
    }
}
```

这样一个接口就已经完成了，启动项目访问`http://localhost:8080/user/1`即可得到如下的结果：
```json
{
"id": 1,
"age": 18,
"birthday": 1601454650860,
"name": "不才陈某"
}
```

### 如何自定义tomcat的端口？
Spring Boot其实默认内嵌了Tomcat，当然默认的端口号也是`8080`，如果需要修改的话，只需要在配置文件中添加如下一行配置即可:
```properties
server.port=9090
```

### 如何自定义项目路径？
在配置文件中添加如下配置即可：
```properties
server.servlet.context-path=/springboot01
```

以上的端口和项目路径改了之后，只需要访问`http://localhost:9090/springboot01/user/1`即可。


## JSON格式化
在前后端分离的项目中大部分的接口基本都是返回JSON字符串，因此对返回的JSON也是需要定制一下，比如**日期的格式**，**NULL值是否返回**等等内容。

Spring Boot默认是使用Jackson对返回结果进行处理，在引入WEB启动器的时候会引入相关的依赖，如下图：

![](https://img.java-family.cn/Spring%20Boot%E7%AC%AC%E4%BA%94%E5%BC%B9%EF%BC%8Cweb%E5%88%9D%E5%85%A5%E9%97%A8/1.png)

**同样是引入了一个启动器，则意味着我们既可以在配置文件中修改配置，也可以在配置类中重写其中的配置。JackSon的自动配置类是`JacksonAutoConfiguration`**
  
### 日期格式的设置

上面的例子中日期的返回结果其实是一个时间戳，那么我们需要返回格式为`yyyy-MM-dd HH:mm:ss`。

可以在配置文件`application.properties`中设置指定的格式，这属于**全局配置**，如下：
```properties
spring.jackson.date-format= yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone= GMT+8
```

也可以在实体属性中标注`@JsonFormat`这个注解，属于局部配置，会覆盖全局配置，如下：
```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm",timezone = "GMT+8")
    private Date birthday;
```

上述日期格式配置完成之后返回的就是指定格式的日期，如下：
```json
{
"id": "1",
"age": 18,
"birthday": "2020-09-30 17:21",
"name": "不才陈某"
}
```

### 其他属性的配置
Jackson还有很多的属性可以配置，这里就不再一一介绍了，所有的配置前缀都是`spring.jackson`。

### 如何在配置类配置？
前面说过在引入WEB模块的时候还引入了JackSon的启动器，这是个好东西，这也是Spring Boot的好处之一，自动配置类中所需的一些配置既可以在全局配置文件`application.properties`中配置也可以在配置类中重新注入某个Bean而达到修改默认配置的效果。

在JackSon自动配置类`JacksonAutoConfiguration`中有如下一段代码：
```java
    @Bean
		@Primary
		@ConditionalOnMissingBean
		ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
			return builder.createXmlMapper(false).build();
		}
```

这一段代码可能初学者比较懵逼了，什么意思呢？别着急，`@Bean`这个注解无非就是注入一个Bean到IOC容器中，`@Primary`这个注解自不用说了，剩下的就是`@ConditionalOnMissingBean`这个注解了，什么意思呢？

**其实仔细研究过Spring Boot的源码的朋友都知道，类似这种`@Conditionalxxx`的注解还有很多，这里就不再深入讲了，后期的文章会介绍。**

`@ConditionalOnMissingBean`这个注解的意思很简单，就是当IOC容器中没有指定Bean的时候才会注入，言下之意就是当容器中不存在`ObjectMapper `这个Bean会使用这里生成的，类似于一种生效的条件。

言外之意就是只需要自定义一个`ObjectMapper`然后注入到IOC容器中，那么这个自动配置类`JacksonAutoConfiguration`中注入的将会失效，也就达到了覆盖的作用了。

因此只需要定义一个配置类，注入`ObjectMapper`即可，如下：
```java
/**
 * 自定义jackson序列化与反序列规则，增加相关格式（全局配置）
 */
@Configuration
public class JacksonConfig {
    @Bean
    @Primary
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        builder.locale(Locale.CHINA);
        builder.timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()));
        builder.simpleDateFormat(DatePattern.NORM_DATETIME_PATTERN);
        builder.modules(new CustomTimeModule());

        ObjectMapper objectMapper = builder.createXmlMapper(false).build();

        objectMapper.setSerializationInclusion(JsonInclude.Include.NON_EMPTY);

        //遇到未知属性的时候抛出异常，//为true 会抛出异常
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        // 允许出现特殊字符和转义符
        objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
        // 允许出现单引号
        objectMapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);


        objectMapper.registerModule(new CustomTimeModule());

        return objectMapper;
    }

}
```

上面只是个例子，关于`ObjectMapper`中的一些内容感兴趣的可以自己查查相关资料。

## 总结
这篇文章算是WEB开发的入门，介绍了如何定义接口，返回JSON如何定制等内容，如果觉得有所收获点点关注在看分享一波！！！






