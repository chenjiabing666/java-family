**大家好，我是不才陈某~**

前几天有个大兄弟问了我一个问题，注册中心要集成SpringCloud，想实现SpringCloud的负载均衡，需要实现哪些接口和规范。

![](https://img.java-family.cn/20230515095354image-20230515095354012.png)

既然这个兄弟问到我了，而我又刚好知道，这不得好好写一篇文章来回答这个问题，虽然在后面的聊天中我已经回答过了。

![](https://img.java-family.cn/20230515095413image-20230515095413448.png)

接下来本文就来探究一下Nacos、OpenFeign、Ribbon、loadbalancer等组件协调工作的原理，知道这些原理之后，就知道应该需要是实现哪些接口了。

再多说一句，本文并没有详细地深入剖析各个组件的源码，如果有感兴趣的兄弟可以从公众号后台菜单栏中的文章分类中查看我之前写的关于Nacos、OpenFeign、Ribbon源码剖析的文章。

## Nacos

Nacos是什么，官网中有这么一段话

![](https://img.java-family.cn/20230515095912image-20230515095912219.png)

这一段话说的直白点就是Nacos是一个注册中心和配置中心！

在Nacos中有客户端和服务端的这个概念

![](https://img.java-family.cn/20230515095918image-20230515095918266.png)

- 服务端需要单独部署，用来保存服务实例数据的
- 客户端就是用来跟服务端通信的SDK，支持不同语言

当需要向Nacos服务端注册或者获取服务实例数据的时候，只需要通过Nacos提供的客户端SDK就可以了，就像下面这样：

引入依赖

```xml
<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
    <version>1.4.4</version>
</dependency>
```

示例代码

```java
Properties properties = new Properties();
properties.setProperty("serverAddr", "localhost");
properties.setProperty("namespace", "8848");

NamingService naming = NamingFactory.createNamingService(properties);

//服务注册，注册一个order服务，order服务的ip是192.168.2.100，端口8080
naming.registerInstance("order", "192.168.2.100", 8080);

//服务发现，获取所有的order服务实例
List<Instance> instanceList = naming.selectInstances("order", true);
```

当服务注册到Nacos服务端的时候，在服务端内部会有一个集合去存储服务的信息

![](https://img.java-family.cn/20230515095927image-20230515095926906.png)

这个集合在注册中心界中有个响亮的名字，**服务注册表**。

### 如何进行服务自动注册？

用过SpringCloud的小伙伴肯定知道，在项目启动的时候服务能够自动注册到服务注册中心，并不需要手动写上面那段代码，那么服务自动注册是如何实现的呢？

#### 服务自动注册三板斧

SpringCloud本身提供了一套服务自动注册的机制，或者说是约束，其实就是三个接口，只要注册中心实现这些接口，就能够在服务启动时自动注册到注册中心，而这三个接口我称为服务自动注册三板斧。

##### 服务实例数据封装--Registration

Registration是SpringCloud提供的一个接口，继承了ServiceInstance接口

![](https://img.java-family.cn/20230515095934image-20230515095934535.png)

![](https://img.java-family.cn/20230515095950image-20230515095950264.png)

从ServiceInstance的接口定义可以看出，这是一个服务实例数据的封装，比如这个服务的ip是多少，端口号是多少。

所以Registration就是当前服务实例数据封装，封装了当前服务的所在的机器ip和端口号等信息。

Nacos既然要整合SpringCloud，自然而然也实现了这个接口

![](https://img.java-family.cn/20230515095958image-20230515095958104.png)

这样当前服务需要被注册到注册中心的信息就封装好了。

##### 服务注册--ServiceRegistry

ServiceRegistry也是个接口，泛型就是上面提到的服务实例数据封装的接口

![](https://img.java-family.cn/20230515100008image-20230515100008640.png)

这个接口的作用就是把上面封装的当前服务的数据Registration注册通过`register`方法注册到注册中心中。

Nacos也实现了这个接口。

![](https://img.java-family.cn/20230515100015image-20230515100014958.png)

并且核心的注册方法的实现代码跟前面的demo几乎一样

![](https://img.java-family.cn/20230515100022image-20230515100022614.png)

##### 服务自动注册--AutoServiceRegistration

![](https://img.java-family.cn/20230515100030image-20230515100029790.png)

AutoServiceRegistration是一个标记接口，所以本身没有实际的意义，仅仅代表了自动注册的意思。

AutoServiceRegistration有个抽象实现AbstractAutoServiceRegistration

![](https://img.java-family.cn/20230515100037image-20230515100037349.png)

AbstractAutoServiceRegistration实现了ApplicationListener，监听了WebServerInitializedEvent事件。

WebServerInitializedEvent这个事件是SpringBoot在项目启动时，当诸如tomcat这类Web服务启动之后就会发布，注意，只有在Web环境才会发布这个事件。

![](https://img.java-family.cn/20230515100046image-20230515100046082.png)

ServletWebServerInitializedEvent继承自WebServerInitializedEvent。

所以一旦当SpringBoot项目启动，tomcat等web服务器启动成功之后，就会触发AbstractAutoServiceRegistration监听器的执行。

最终就会调用ServiceRegistry注册Registration，实现服务自动注册

![](https://img.java-family.cn/20230515100055image-20230515100055258.png)

Nacos自然而然也继承了AbstractAutoServiceRegistration

![](https://img.java-family.cn/20230515100104image-20230515100103869.png)

对于Nacos而言，就将当前的服务注册的ip和端口等信息，就注册到了Nacos服务注册中心。

所以整个注册流程就可以用这么一张图概括

![](https://img.java-family.cn/20230515100110image-20230515100110305.png)

当然，不仅仅是Nacos是这么实现的，常见的比如Eureka、Zookeeper等注册中心在整合SpringCloud都是实现上面的三板斧。

![](https://img.java-family.cn/20230515100119image-20230515100119018.png)

## Ribbon

讲完了SpringCloud环境底下是如何自动注册服务到注册中心的，下面来讲一讲Ribbon。

我们都知道，Ribbon是负载均衡组件，他的作用就是从众多的服务实例中根据一定的算法选择一个服务实例。

但是有个疑问，服务实例的数据都在注册中心，Ribbon是怎么知道的呢？？？

答案其实很简单，那就是需要注册中心去主动**适配**Ribbon，只要注册中心去适配了Ribbon，那么Ribbon自然而然就知道服务实例的数据了。

Ribbon提供了一个获取服务实例的接口，叫ServerList

![](https://img.java-family.cn/20230515100129image-20230515100128940.png)

接口中提供了两个方法，这两个方法在众多的实现中实际是一样的，并没有区别。

当Ribbon通过ServerList获取到服务实例数据之后，会基于这些数据来做负载均衡的。

Nacos自然而然也实现了ServerList接口，为Ribbon提供Nacos注册中心中的服务数据。

![](https://img.java-family.cn/20230515100138image-20230515100137978.png)

这样，Ribbon就能获取到了Nacos服务注册中心的数据。

同样地，除了Nacos之外，Eureka、Zookeeper等注册中心也都实现了这个接口。

![](https://img.java-family.cn/20230515100144image-20230515100144224.png)

到这，其实就明白了Ribbon是如何知道注册中心的数据了，需要注册中心来适配。

在这里插个个人的看法，其实我觉得Ribbon在适配SpringCloud时对获取服务实例这块支持封装的不太好。

![](https://img.java-family.cn/20230515100152image-20230515100152066.png)

因为SpringCloud本身就是一套约束、规范，只要遵守这套规范，那么就可以实现各个组件的替换，这就是为什么换个注册中心只需要换个依赖，改个配置文件就行。

而Ribbon本身是一个具体的负载均衡组件，注册中心要想整合SpringCloud，还得需要单独去适配Ribbon，有点违背了SpringCloud约束的意义。

就类似mybatis一样，mybatis依靠jdbc，但是mybatis根本不关心哪个数据库实现的jdbc。

真正好的做法是Ribbon去适配SpringCloud时，用SpringCloud提供的api去获取服务实例，这样不同的注册中心只需要适配这个api，无需单独适配Ribbon了。

而SpringCloud实际上是提供了这么一个获取服务实例的api，DiscoveryClient

![](https://img.java-family.cn/20230515100158image-20230515100158522.png)

通过DiscoveryClient就能够获取到服务实例，当然也是需要不同注册中心的适配。

![](https://img.java-family.cn/20230515100206image-20230515100205953.png)

随着Ribbon等组件停止维护之后，SpringCloud官方自己也搞了一个负载均衡组件`loadbalancer`，用来平替Ribbon。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

这个组件底层在获取服务实例的时候，就是使用的DiscoveryClient。

![](https://img.java-family.cn/20230515100212image-20230515100212478.png)

所以对于`loadbalancer`这个负载均衡组价来说，注册中心只需要实现DiscoveryClient之后就自然而然适配了`loadbalancer`。

## OpenFeign

OpenFeign是一个rpc框架，当我们需要调用远程服务的时候，只需要声明个接口就可以远程调用了，就像下面这样

![](https://img.java-family.cn/20230515100220image-20230515100220182.png)

听上去很神奇，其实本质上就是后面会为接口创建一个动态代理对象，解析类上，方法上的注解。

当调用方法的时候，会根据方法上面的参数拼接一个http请求地址，这个地址的格式是这样的`http://服务名/接口路径`。

比如，上面的例子，当调用`saveOrder`方法的时候，按照这种规律拼出的地址就是这样的 `http://order/order`，第一个order是服务名，第二个order是PostMapping注解上面的。

但是由于只知道需要调用服务的服务名，不知道服务的ip和端口，还是无法调用远程服务，这咋办呢？

这时就轮到Ribbon登场了，因为Ribbon这个大兄弟知道服务实例的数据。

于是乎，OpenFeign就对Ribbon说，兄弟，你不是可以从注册中心获取到order服务所有服务实例数据么，帮我从这些服务实例数据中找一个给我。

![](https://img.java-family.cn/20230515100227image-20230515100226946.png)

于是Ribbon就会从注册中心获取到的服务实例中根据负载均衡策略选择一个服务实例返回给OpenFeign。

OpenFeign拿到了服务实例，此时就获取到了服务所在的ip和端口，接下来就会重新构建请求路径，将路径中的服务名替换成ip和端口，代码如下

![](https://img.java-family.cn/20230515100233image-20230515100232986.png)

- Server就是服务实例信息的封装
- orignal就是原始的url，就是上面提到的，`http://order/order`

假设获取到的orde服务所在的ip和端口分别是`192.168.2.100`和`8080`，最终重构后的路径就是`http://192.168.2.100:8080/order`，之后OpenFeign就可以发送http请求了。

至于前面提到的`loadbalancer`，其实也是一样的，他也会根据负载均衡算法，从DiscoveryClient获取到的服务实例中选择一个服务实例给OpenFeign，后面也会根据服务实例重构url，再发送http请求。

![](https://img.java-family.cn/20230515100240image-20230515100240598.png)

## 总结

到这，就把Nacos、OpenFeign、Ribbon、loadbalancer等组件协调工作的原理讲完了，其实就是各个组件会预留一些扩展接口，这也是很多开源框架都会干的事，当第三方框架去适配的，只要实现这些接口就可以了。

最后画一张图来总结一下上述组价的工作的原理。

![](https://img.java-family.cn/20230515100248image-20230515100248565.png)

最后再小小地说一句，Nacos、OpenFeign、Ribbon源码剖析的文章，可以从公众号后台菜单栏中的文章分类中查看。

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

另外陈某的[知识星球](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247524437&idx=1&sn=32699e9afede86fe52136c38f844c620&chksm=fcf7bf98cb80368e5eff366916e6b251fa7ca7526b44202d59032863d89f353545f0c0ebfff2&token=339165402&lang=zh_CN#rd)开通了，公众号回复关键词：**知识星球** 获取限量**20元**优惠券加入只需**109**元，星球回馈的价值巨大，目前更新了**Spring全家桶实战系列**、**亿级数据分库分表实战**、**DDD微服务实战专栏**、**我要进大厂、Spring，Mybatis等框架源码、架构实战22讲、精尽RocketMQ**等....每增加一个专栏价格将上涨20元

![](https://img.java-family.cn/20230515100340image-20230515100339945.png)

关注公众号：【码猿技术专栏】，公众号内有超赞的粉丝福利，回复：加群，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！