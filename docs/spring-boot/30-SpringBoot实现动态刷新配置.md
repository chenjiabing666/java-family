## 前言
对于微服务而言配置本地化是个很大的鸡肋，不可能每次需要改个配置都要重新把服务重新启动一遍，因此最终的解决方案都是将配置外部化，托管在一个平台上达到不用重启服务即可`一次修改多处生效`的目的。

但是对于单体应用的Spring Boot项目而言，动态刷新显然是有点多余，反正就一个服务，改下重启不就行了？

然而在某些特殊的场景下还是必须用到动态刷新的，如下：
1. `添加数据源`：对接某个第三方平台的时候，你不可能每次添加一个数据源都要重启下服务
2. `固化的对接`：大量的固定对接方式，只是其中的某个固定的代码段不同，比如提供视图中的字段不同，接口服务中字段不同等情况。

当然以上列举的两种场景每个公司都有不同的解决方案，这里不做深究。

[如何让Spring Boot 的配置 “动” 起来？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247491723&idx=1&sn=4f335dfab579aac6cd40455d88f74fdb&chksm=fcf73f46cb80b6503898214461b5fe173319e46da76d8b5de531db6588a30cb0ccadf53fbdb6&token=381464093&lang=zh_CN#rd)

## 微服务下有哪几种主流的方案？

微服务下的动态配置中心有三种主流的方式，如下图：

![主流的配置中心](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/1.png)

上图中的三种配置中心方案可以说是现在企业中使用率最高的，分别是：
1. **Nacos**:阿里巴巴的最近开源的项目，这个家伙很牛逼，一个干掉了`Eureka`(停更)和`Config+Bus`，既能作为配置中心也能作为注册中心，并且有自己的独立的 管理平台，可以说是现在最主流的一种。

2. **Config+Bus**：早期在用的微服务配置中心，可以依托`GitHub`管理微服务的配置文件，这种现在也是有不少企业在用，但是需要自己独立部署一个微服务，和`Nacos`相比逊色了不少。

3. **Apollo**：携程开源项目Apollo，这个也是不少企业在用，陈某了解的不多，有兴趣的可以深入研究下。


## 针对Spring Boot 适用的几种方案？

其实上述三种都可以在Spring Boot项目中适配，但是作为单体应用有些重了，下面作者简单的介绍两种可用的方案。

### Spring Boot+Nacos（不推荐）

不得不说阿里巴巴确实挺有野心，阿里要做的其实是一个微服务生态，Nacos不仅仅可以作为Spring Cloud的配置和注册中心，也适配了Dubbo、K8s，官方文档中对于如何适配都做了详细的介绍，作者 这里就不再详细介绍了，如下图：

![Nacos.io](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/2.png)

> 当然Nacos对Spring、Spring Boot 项目同样适用。

如何使用呢？这里作者只提供下思路，不做过多的深究，这篇在作者下个专栏**Spring Cloud 进阶**会详细介绍：
1. 下载对应版本的Nacos，启动项目，访问`http://localhost:8848`进入Nacos的管理界面；

2. Spring Boot 项目引入Nacos的配置依赖`nacos-config-spring-boot-starter`，配置Nacos管理中心的地址。

3. `@NacosPropertySource`、`@NacosValue`两个注解结合完成。
  - `@NacosPropertySource`：指定配置中心的`dataId`，和是否自动刷新
  - `@NacosValue`替代`@Value`注解完成属性的自动装配
4. 如果公司项目做了后台管理，则可以直接调用Nacos开放的API修改对应配置的值（替代了Nacos管理界面的手动操作），API的地址：https://nacos.io/zh-cn/docs/open-api.html

此种方案虽说可以实现配置的动态刷新，但是还要集成Nacos，启动一个Nacos的服务，完全是有点大材小用了，实际项目中不推荐使用。

### Spring Boot+Config+actuator（推荐）

此种方案实际使用的是Config配置中心，但是不像Nacos那般重，完全适用于单体应用的SpringBoot项目，只需要做小部分的更改即可达到效果。

#### 方案一（不推荐）

1. 添加Config的依赖，如下：
```xml
<!-- springCloud的依赖-->
<dependencyManagement>
    <dependencies>
        <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR3</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    </dependencies>
</dependencyManagement>

<!-- config的依赖-->
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>

  <!-- actuator的依赖-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

2. 配置文件中暴露Spring Boot的端点，如下：
```properties
management.endpoints.web.exposure.include=*
```

3. 配置文件中新增三个属性配置：
```properties
config.version=22
config.app.name=dynamic-project
config.platform=mysql
```

4. 结合`@RefreshScope`注解动态刷新，写个Controller，如下：

```java
@RestController
//@RefreshScope该注解必须标注，否则无法完成动态更新
@RefreshScope
public class DynamicConfigController {
    @Value("${config.version}")
    private String version;

    @Value("${config.app.name}")
    private String appName;

    @Value("${config.platform}")
    private String platform;


    @GetMapping("/show/version")
    public String test(){
        return "version="+version+"-appName="+appName+"-platform="+platform;
    }
```

4. 启动项目测试，浏览器访问`http://localhost:8080/show/version`，返回信息如下图：

![](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/3.png)

5. 修改`target`目录下的配置文件，如下：
```properties
config.version=33
config.app.name=dynamic-project
config.platform=ORACLE
```

6. POST请求`http://localhost:8080/actuator/refresh`接口，手动刷新下配置（必须，否则不能自动刷新）

7. 浏览器再次输入`http://localhost:8080/show/version`，结果如下图：

![](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/4.png)

可以看到，配置已经自动修改了，结束。

##### 方案二（推荐）

看到了方案一觉得如何？是不是有点鸡肋了

> 第一个问题：为什么还要调用一次手动刷新呢？

> 第二个问题：只能手动的在配置文件中改吗？如果想在后台管理系统改怎么办？

想要解决上述两个问题还是要看下`Config`的源码，代码关键部分在`org.springframework.cloud.context.refresh.ContextRefresher#refresh()`方法中，如下图：

![](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/5.png)

因此只需要在修改属性之后调用下`ContextRefresher#refresh()`（异步，避免一直阻塞等待）方法即可。

为了方便测试，我们自己手动写一个refresh接口，如下：
```java
@GetMapping("/show/refresh")
    public String refresh(){
        //修改配置文件中属性
        HashMap<String, Object> map = new HashMap<>();
        map.put("config.version",99);
        map.put("config.app.name","appName");
        map.put("config.platform","ORACLE");
        MapPropertySource propertySource=new MapPropertySource("dynamic",map);
        //将修改后的配置设置到environment中
        environment.getPropertySources().addFirst(propertySource);
        //异步调用refresh方法，避免阻塞一直等待无响应
        new Thread(() -> contextRefresher.refresh()).start();
        return "success";
    }
```

> 上述代码中作者只是手动设置了配置文件中的值，实际项目中可以通过持久化的方式从数据库中读取配置刷新。

下面我们测试看看，启动项目，访问`http://localhost:8080/show/version`，发现是之前配置在`application.properties`中的值，如下图：

![](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/3.png)

调用`refresh`接口：`http://localhost:8080/show/refresh`重新设置属性值；

再次调用`http://localhost:8080/show/version`查看下配置是否修改了，如下图：

![](https://img.java-family.cn/%E5%8A%A8%E6%80%81%E5%88%B7%E6%96%B0%E9%85%8D%E7%BD%AE/6.png)

从上图可以发现，配置果然修改了，达到了动态刷新的效果。


## 总结

本文从微服务的配置中心介绍到Spring Boot 搭建简易的配置中心，详细介绍了几种可行性的方案，作者强力推荐最后一种方案，简化版的`Config`，完全适用于单体应用。

> 项目源码已经上传，有需要的回复关键词`008`获取。



























