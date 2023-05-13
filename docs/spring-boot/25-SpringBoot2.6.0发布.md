
本来预计12月份中旬发布的**Spring Boot 2.6.0** 版本硬是提前了一个月，当然陈某也是时刻关注Spring Boot 的每一个的版本迭代，今天给大家分享一下**Spring Boot 2.6.0** 更新了什么？

## 1、默认禁止了循环依赖

循环依赖大家都知道，也被折磨过，这下**2.6.0**的版本默认禁止了循环依赖，如果程序中出现循环依赖就会报错。

![](D:\BlogImage\MQ\40.png)

当然并没有一锤子打死，也提供了开启允许循环依赖的配置，只需要在配置文件中开启即可：

```yaml
spring:
  main:
    allow-circular-references: true
```

## 2、支持自定义脱敏规则

Spring Boot 现在可以清理 `/env` 和 `/configprops` 端点中存在的敏感值。

自定义**SanitizingFunction**类型的**Bean**即可实现。

```java
@Bean
public SanitizingFunction mobileSanitizingFunction() {
    return data -> {
		PropertySource<?> propertySource = data.getPropertySource();
        if (propertySource.getName().contains("redis.properties")) {
            if (data.getKey().equals("redis.mobile")) {
                return data.withValue(SANITIZED_VALUE);
            }
        }
        return data;
    };
}
```

关于脱敏陈某之前写过一篇文章，已经非常详细了：[Springboot 日志、配置文件、接口数据如何脱敏？老鸟们都是这样玩的！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247495798&amp;idx=1&amp;sn=95f537d8e82823b668b1c7a2d6842827&amp;chksm=fcf72fbbcb80a6ad42a12221d3b075670deb5014389db03b7cb8a9444f3776c54d52b98ba404&token=1286998820&lang=zh_CN#rd)

## 3、Redis自动开启连接池

这个版本之前Redis连接池需要开发主动开启，但是这个版本默认是开启的。

如果需要关闭一样是提供了配置，如下：

**1、jedis连接池关闭**：

```properties
spring.redis.jedis.pool.enabled = false
```

**2、lettuce连接池关闭**：

```properties
spring.redis.lettuce.pool.enabled = false 
```

## 4、响应式应用服务器会话属性

响应式应用服务器支持的会话属性已在此版本中扩展。

以前是在 **spring.webflux.session**下，现在在 **server.reactive.session** 下，并且提供与 servlet 版本相同的属性。



## 5、Maven构建信息属性排除

现在可以从 Spring Boot Maven 或 Gradle 插件生成的 build-info.properties 文件中排除特定属性。

比如，排除 Maven 的 version 属性：

```xml
<configuration>	
    <excludeInfoProperties>		
        <excludeInfoProperty>version</excludeInfoProperty>	
    </excludeInfoProperties>
</configuration>
```

## 6、支持使用WebTestClient来测试Spring MVC

开发人员可以使用 **WebTestClient** 在模拟环境中测试程序，只需要在Mock环境中使用 **@AutoConfigureMockMvc**注释，就可以轻松注入 **WebTestClient**。，省去编写测试程序。

## 7、支持 Log4j2 复合配置

现在支持 Log4j2 的复合配置，可以通过 **logging.log4j2.config.override** 参数来指定覆盖主日志配置文件的其他日志配置文件。

## 总结

以上陈某只是总结了比较重要的几点，这个版本变动还是有些大的，具体细节可以看官方文档：https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes

**你们现在都用的哪个版本？欢迎评论区留言！**

