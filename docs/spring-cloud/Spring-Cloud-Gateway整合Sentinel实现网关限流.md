**大家好，我是不才陈某~**

这是[《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114&scene=126#wechat_redirect)第**八**篇文章，往期文章如下：

- [五十五张图告诉你微服务的灵魂摆渡者Nacos究竟有多强？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247493854&idx=1&sn=4b3fb7f7e17a76000733899f511ef915&scene=21#wechat_redirect)
- [openFeign夺命连环9问，这谁受得了？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247496653&idx=1&sn=7185077b3bdc1d094aef645d677ec472&scene=21#wechat_redirect)
- [阿里面试这样问：Nacos、Apollo、Config配置中心如何选型？这10个维度告诉你！](https://mp.weixin.qq.com/s/S_8HQYHOG624Vzeu94CFSA)
- [阿里面试败北：5种微服务注册中心如何选型？这几个维度告诉你！](https://mp.weixin.qq.com/s/YKzYdMu-7nwszEf9-1M3Uw)
- [阿里限流神器Sentinel夺命连环 17 问？](https://mp.weixin.qq.com/s/Q7Xv8cypQFrrOQhbd9BOXw)
- [对比7种分布式事务方案，还是偏爱阿里开源的Seata，真香！(原理+实战)](https://mp.weixin.qq.com/s/sXVSFqq2UZ6Pwwt7vx7vIA)
- [Spring Cloud Gateway夺命连环10问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247499894&idx=1&sn=f1606e4c00fd15292269afe052f5bca2&chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&token=1887105114&lang=zh_CN#rd)

前一篇文章介绍了[Spring Cloud Gateway]((https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247499894&idx=1&sn=f1606e4c00fd15292269afe052f5bca2&chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&token=1887105114&lang=zh_CN#rd))的一些基础知识点，今天陈某就来唠一唠网关层面如何做限流？

文章目录如下：

![](https://img.java-family.cn/gateway/25.png)



## 网关如何限流？

Spring Cloud Gateway本身自带的限流实现，过滤器是`RequestRateLimiterGatewayFilterFactory`，不过这种上不了台面的就不再介绍了，有兴趣的可以实现下。

今天的重点是集成阿里的**Sentinel**实现网关限流，sentinel有不懂的可以看陈某的文章：[阿里限流神器Sentinel夺命连环 17 问？](https://mp.weixin.qq.com/s/Q7Xv8cypQFrrOQhbd9BOXw)

  从1.6.0版本开始，Sentinel提供了SpringCloud Gateway的适配模块，可以提供两种资源维度的限流：

- **route维度**：即在配置文件中配置的路由条目，资源名为对应的`routeId`，这种属于粗粒度的限流，一般是对某个微服务进行限流。
- **自定义API维度**：用户可以利用Sentinel提供的API来自定义一些API分组，这种属于细粒度的限流，针对某一类的uri进行匹配限流，可以跨多个微服务。

> sentinel官方文档：https://github.com/alibaba/Sentinel/wiki/%E7%BD%91%E5%85%B3%E9%99%90%E6%B5%81

Spring Cloud Gateway集成Sentinel实现很简单，这就是阿里的魅力，提供简单、易操作的工具，让程序员专注于业务。

### 新建项目

新建一个`gateway-sentinel9026`模块，添加如下依赖：

```xml
<!--nacos注册中心-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--spring cloud gateway-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!--    spring cloud gateway整合sentinel的依赖-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>

    <!--    sentinel的依赖-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
```

> **注意**：这依然是一个网关服务，不要添加WEB的依赖

### 配置文件

配置文件中主要指定以下三种配置：

- nacos的地址
- sentinel控制台的地址
- 网关路由的配置

配置如下：

```yaml
spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      transport:
        ## 指定控制台的地址，默认端口8080
        dashboard: localhost:8080
    nacos:
      ## 注册中心配置
      discovery:
        # nacos的服务地址，nacos-server中IP地址:端口号
        server-addr: 127.0.0.1:8848
    gateway:
      ## 路由
      routes:
        ## id只要唯一即可，名称任意
        - id: gateway-provider
          uri: lb://gateway-provider
          ## 配置断言
          predicates:
            ## Path Route Predicate Factory断言，满足/gateway/provider/**这个请求路径的都会被路由到http://localhost:9024这个uri中
            - Path=/gateway/provider/**
```

上述配置中设置了一个路由`gateway-provider`，只要请求路径满足`/gateway/provider/**`都会被路由到`gateway-provider`这个服务中。

### 限流配置

经过上述两个步骤其实已经整合好了Sentinel，此时访问一下接口：http://localhost:9026/gateway/provider/port

然后在sentinel控制台可以看到已经被监控了，监控的路由是`gateway-provider`，如下图：

![](https://img.java-family.cn/gateway/17.png)

此时我们可以为其新增一个route维度的限流，如下图：

![](https://img.java-family.cn/gateway/18.png)

上图中对`gateway-provider`这个路由做出了限流，QPS阈值为1。

此时快速访问：http://localhost:9026/gateway/provider/port，看到已经被限流了，如下图：

![](https://img.java-family.cn/gateway/19.png)

以上route维度的限流已经配置成功，小伙伴可以自己照着上述步骤尝试一下。

API分组限流也很简单，首先需要定义一个分组，**API管理-> 新增API分组**，如下图：

![](https://img.java-family.cn/gateway/20.png)

匹配模式选择了精确匹配（还有前缀匹配，正则匹配），因此只有这个uri：`http://xxxx/gateway/provider/port`会被限流。

第二步需要对这个分组添加流控规则，**流控规则->新增网关流控**，如下图：

![](https://img.java-family.cn/gateway/21.png)

API名称那里选择对应的分组即可，新增之后，限流规则就生效了。

陈某不再测试了，小伙伴自己动手测试一下吧...............

> 陈某这里只是简单的配置一下，至于限流规则持久化一些内容请看陈某的Sentinel文章，这里就不再过多的介绍了。

## 如何自定义限流异常信息？

从上面的演示中可以看到默认的异常返回信息是："Block........."，这种肯定是客户端不能接受的，因此需要定制自己的异常返回信息。

下面介绍两种不同的方式定制异常返回信息，开发中自己选择其中一种。

### 直接配置文件中定制

开发者可以直接在配置文件中直接修改返回信息，配置如下：

```yaml
spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      #配置限流之后，响应内容
      scg:
        fallback:
          ## 两种模式，一种是response返回文字提示信息，
          ## 一种是redirect，重定向跳转，需要同时配置redirect(跳转的uri)
          mode: response
          ## 响应的状态
          response-status: 200
          ## 响应体
          response-body: '{"code": 200,"message": "请求失败，稍后重试！"}'
```

上述配置中`mode`配置的是`response`，一旦被限流了，将会返回`JSON`串。

```json
{
    "code": 200,
    "message": "请求失败，稍后重试！"
}
```



**重定向**的配置如下：

```yaml
spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      #配置限流之后，响应内容
      scg:
        fallback:
          ## 两种模式，一种是response返回文字提示信息，一种是redirect，重定向跳转，需要同时配置redirect(跳转的uri)
          mode: redirect
          ## 跳转的URL
          redirect: http://www.baidu.com
```

一旦被限流，将会直接跳转到：http://www.baidu.com

### 编码定制

这种就不太灵活了，通过硬编码的方式，完整代码如下：

```java
@Configuration
public class GatewayConfig {
    /**
     * 自定义限流处理器
     */
    @PostConstruct
    public void initBlockHandlers() {
        BlockRequestHandler blockHandler = (serverWebExchange, throwable) -> {
            Map map = new HashMap();
            map.put("code",200);
            map.put("message","请求失败，稍后重试！");
            return ServerResponse.status(HttpStatus.OK)
                    .contentType(MediaType.APPLICATION_JSON_UTF8)
                    .body(BodyInserters.fromObject(map));
        };
        GatewayCallbackManager.setBlockHandler(blockHandler);
    }
}
```

两种方式介绍完了，根据业务需求自己选择适合的方式，当然陈某更喜欢第一种，理由：**约定>配置>编码**。



## 网关限流了，服务就安全了吗？

很多人认为只要网关层面做了限流，躲在身后的服务就可以高枕无忧了，你是不是也有这种想法？

很显然这种想法是错误的，复杂的微服务架构一个独立服务不仅仅被一方调用，往往是多方调用，如下图：

![](https://img.java-family.cn/gateway/22.png)

商品服务不仅仅被网关层调用，还被内部订单服务调用，这时候仅仅在网关层限流，那么商品服务还安全吗？

一旦大量的请求订单服务，比如大促秒杀，商品服务不做限流会被瞬间击垮。

因此需要根据公司业务场景对自己负责的服务也要进行限流兜底，最常见的方案：**网关层集群限流+内部服务的单机限流兜底**，这样才能保证不被流量冲垮。



## 总结

文章介绍了Spring Cloud Gateway整合Sentinel对网关层进行限流，以及关于限流的一些思考。如有错误之处，欢迎留言指正。

> 项目源码已经上传Github，公众号【码猿技术专栏】回复关键词：**9528**获取！

## 最后说一句（求关注，别白嫖我）

陈某每一篇原创文章都是精心输出，尤其是《Spring Cloud 进阶》专栏的文章，知识点太多，要想讲的细，必须要花很多时间准备，从知识点到源码demo。

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：【码猿技术专栏】，公众号内有超赞的粉丝福利，回复：加群，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！



## 文末福利（认真看抽奖方式哦）

陈某一直以来也没送过什么福利给各位小伙伴，这不最近联系了出版社要了几本**《Spring Cloud Alibaba 微服务实战》**，刚好陈某最近也在连载[《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114&scene=126#wechat_redirect)专栏，那么今天  **免费送给大家几本** ！

![](https://img.java-family.cn/gateway/1.jpg)



本书内容丰富，案例通俗易懂，几乎涵盖了目前Spring Cloud的全部热门组件，特别适合想要了解Spring Cloud热门组件以及想搭建微服务系统的读者阅读。

**抽奖方式**：在此文留言，然后将文章转发至你朋友圈，让你朋友们来给你的留言点赞，留言点赞数量靠前的获赠此书。**（此文阅读量越高书送的越多，所以一定要多多转发呀！）**

**截止时间**：2021/11/4 24:00:00

当然，不差钱的小伙伴也可以直接通过下面的小程序直接购买。







