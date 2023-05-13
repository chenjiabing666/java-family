**大家好，我是不才陈某~**

在生产环境中，如何保证在服务升级的时候，不影响用户的体验，这个是一个非常重要的问题。如果在我们升级服务的时候，会造成一段时间内的服务不可用，这就是不够优雅的。

那什么是优雅的呢？主要就是指在服务升级的时候，不中断整个服务，让用户无感知，进而不会影响用户的体验，这就是优雅的。

实际上，优雅下线是目标，而不是手段，它是一个相对的概念，例如 **kill PID** 和 **kill -9 PID** 都是暴力杀死服务，相对于 **kill -9 PID** 来说，**kill PID** 就是优雅的。

但如果单独拿 kill PID 出来说，我们能说它是优雅的下线策略吗？肯定不是啊，就是这个道理。

因此，本文讲述的优雅下线仅能称之为“相对的优雅下线”，但相对于暴力的杀死服务，已经足够优雅了。

> 上篇文章：[聊聊 Spring Cloud 全链路灰度发布 方案~](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247508403&idx=1&sn=be24819cfea40d8c76cc009fb8784e47&chksm=fcf77e7ecb80f768d59716a6af9e6161d994171f08966fbeb7fc0f5b6ac4f0ae5864ac6b781e&token=1010805510&lang=zh_CN#rd)介绍到的灰度发布当然也算是一种优雅下线的方案。



## **方式一：kill PID**

使用方式：kill java 进程 ID。该方式借助的是 SpringBoot 应用的 Shutdown hook，应用本身的下线也是优雅的，但如果你的服务发现组件使用的是 Eureka，那么默认最长会有 90 秒的延迟，其他应用才会感知到该服务下线。

这意味着：该实例下线后的 90 秒内，其他服务仍然可能调用到这个已下线的实例。因此，该方式是不够优雅的。



## **方式二：/shutdown 端点**



SpringBoot 提供了 /shutdown 端点，可以借助它实现优雅停机。



**使用方式：**在想下线应用的 application.yml 中添加如下配置，从而启用并暴露 /shutdown 端点。

```yaml
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include: shutdown
```



发送 POST 请求到 **/shutdown** 端点：**curl -X http://你想停止的服务地址/actuator/shutdown**。

> 该方式本质和方式一是一样的，也是借助 SpringBoot 应用的 Shutdown hook 去实现的。



## **方式三：/pause 端点**



SpringBoot 应用提供了 **/pause** 端点，利用该端点可实现优雅下线。

使用方式：在想下线应用的 **application.yml** 中添加配置，从而启用并暴露 /pause 端点。

```yaml
management:
  endpoint:
    # 启用pause端点
    pause:
      enabled: true
    # 启用restart端点，之所以要启用restart端点，是因为pause端点的启用依赖restart端点的启用
    restart:
      enabled: true
  endpoints:
    web:
      exposure:
        include: pause,restart
```



发送 POST 请求到 /actuator/pause 端点：**curl -X POST http://你想停止的服务实例地址/actuator/pause**。

执行后的效果类似下图：

![](https://img.java-family.cn/%E4%BC%98%E9%9B%85%E4%B8%8B%E7%BA%BF/1.png)

如图所示，该应用在 Nacos 上的状已被标记为下线了，但是应用本身其实依然是可以正常对外服务的。



在 SpringCloud 中，Ribbon 做负载均衡时，只会负载到标记为　UP　的实例上。

利用这两点，你可以：先用　/pause　端点，将要下线的应用标记为　DOWN，但不去真正停止应用；然后过一定的时间（例如 90 秒，或者自己做个监控，看当前实例的流量变成 0 后）再去停止应用，例如　kill　应用。

缺点 & 局限如下图：

![](https://img.java-family.cn/%E4%BC%98%E9%9B%85%E4%B8%8B%E7%BA%BF/2.png)

## **方式四：/service-registry　端点** 



使用方式：在想下线应用的　application.yml　中添加配置，从而暴露　/service-registry　端点。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: service-registry
```



发送 POST 请求到　/actuator/service-registry　端点：

```shell
curl -X "POST" "http://localhost:8000/actuator/service-registry?status=DOWN" \
   -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
```



**实行后的效果类似如下图：**

![](https://img.java-family.cn/%E4%BC%98%E9%9B%85%E4%B8%8B%E7%BA%BF/1.png)



## **优雅的下线方式**

在上文中，我们讲述了四种常见的下线方式，对比来看，方式四是一种比较优雅的下线方式。

在实际项目中，我们可以先使用　**/service-registry**　端点，将服务标记为　**DOWN**，然后监控服务的流量，当流量为 0 时，即可升级该服务。

当然，这里假设我们部署了多个服务实例，当一个服务实例　DOWN　掉之后，其他服务实例仍然是可以提供服务的，如果就部署一台服务的话，那么讨论优不优雅就没那么重要了。

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！
