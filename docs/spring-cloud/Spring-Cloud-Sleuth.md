# 分布式链路追踪组件 Spring Cloud Sleuth

## 前言


今天这篇文章陈某介绍一下链路追踪相关的知识，以**Spring Cloud Sleuth**和**zipkin**这两个组件为主，后续文章介绍另外一种。

文章的目录如下：

![](https://img.java-family.cn/sleuth/19.png)



## 为什么需要链路追踪？

大型分布式微服务系统中，一个系统被拆分成N多个模块，这些模块负责不同的功能，组合成一套系统，最终可以提供丰富的功能。在这种分布式架构中，一次请求往往需要涉及到多个服务，如下图：

![](https://img.java-family.cn/sleuth/1.png)

服务之间的调用错综复杂，对于维护的成本成倍增加，势必存在以下几个问题：

- 服务之间的依赖与被依赖的关系如何能够清晰的看到？
- 出现异常时如何能够快速定位到异常服务？
- 出现性能瓶颈时如何能够迅速定位哪个服务影响的？

为了能够在分布式架构中快速定位问题，分布式链路追踪应运而生。将一次分布式请求还原成调用链路，进行日志记录，性能监控并将一次分布式请求的调用情况集中展示。比如各个服务节点上的耗时、请求具体到达哪台机器上、每个服务节点的请求状态等等。



## 常见的链路追踪技术有哪些？

市面上有很多链路追踪的项目，其中也不乏一些优秀的，如下：

- **cat**：由大众点评开源，基于Java开发的实时应用监控平台，包括实时应用监控，业务监控 。 集成方案是通过代码埋点的方式来实现监控，比如： 拦截器，过滤器等。 对代码的侵入性很大，集成成本较高，风险较大。
  
- **zipkin**：由Twitter公司开源，开放源代码分布式的跟踪系统，用于收集服务的定时数据，以解决微服务架构中的延迟问题，包括：数据的收集、存储、查找和展现。该产品结合`spring-cloud-sleuth`使用较为简单， 集成很方便， 但是功能较简单。

- **pinpoint**：韩国人开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多种插件，UI功能强大，接入端无代码侵入

- **skywalking**：SkyWalking是本土开源的基于字节码注入的调用链分析，以及应用监控分析工具。特点是支持多种插件，UI功能较强，接入端无代码侵入。目前已加入Apache孵化器。

- **Sleuth**：SpringCloud 提供的分布式系统中链路追踪解决方案。很可惜的是阿里系并没有链路追踪相关的开源项目，我们可以采用**Spring Cloud Sleuth+Zipkin**来做链路追踪的解决方案。



## Spring Cloud Sleuth是什么？

Spring Cloud Sleuth实现了一种分布式的服务链路跟踪解决方案，通过使用Sleuth可以让我们快速定位某个服务的问题。简单来说，Sleuth相当于调用链监控工具的客户端，集成在各个微服务上，负责产生调用链监控数据。

> Spring Cloud Sleuth只负责产生监控数据，通过日志的方式展示出来，并没有提供可视化的UI界面。

学习Sleuth之前必须了解它的几个概念：

- **Span**：基本的工作单元，相当于链表中的一个节点，通过一个唯一ID标记它的开始、具体过程和结束。我们可以通过其中存储的开始和结束的时间戳来统计服务调用的耗时。除此之外还可以获取事件的名称、请求信息等。

- **Trace**：一系列的Span串联形成的一个树状结构，当请求到达系统的入口时就会创建一个唯一ID（traceId），唯一标识一条链路。这个traceId始终在服务之间传递，直到请求的返回，那么就可以使用这个traceId将整个请求串联起来，形成一条完整的链路。

- **Annotation**：一些核心注解用来标注微服务调用之间的事件，重要的几个注解如下：
  - **cs(Client Send)**：客户端发出请求，开始一个请求的生命周期
  - **sr（Server Received）**：服务端接受请求并处理；**sr-cs = 网络延迟 = 服务调用的时间**
  - **ss（Server Send）**：服务端处理完毕准备发送到客户端；**ss - sr = 服务器上的请求处理时间**
  - **cr（Client Reveived）**：客户端接受到服务端的响应，请求结束； **cr - sr = 请求的总时间**
  



## Spring Cloud 如何整合Sleuth？

整合Spring Cloud Sleuth其实没什么的难的，在这之前需要准备以下三个服务：

- **gateway-sleuth9031**：作为网关服务
- **sleuth-product9032**：商品微服务
- **sleuth-order9033**：订单微服务

三个服务的调用关系如下图：

![](https://img.java-family.cn/sleuth/2.png)

客户端请求网关发起查询订单的请求，网关路由给订单服务，订单服务获取订单详情并且调用商品服务获取商品详情。



### 添加依赖

在父模块中添加sleuth依赖，如下：

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

以上只是Spring Cloud Sleuth的依赖，还有**Nacos**，**openFeign**的依赖这里就不再详细说了，有不清楚的可以结合陈某前面几篇文章和案例源码补漏一下。

### 调整日志级别

由于sleuth并没有UI界面，因此需要调整一下日志级别才能在控制台看到更加详细的链路信息。

在三个服务的配置文件中添加以下配置：

```yaml
## 设置openFeign和sleuth的日志级别为debug，方便查看日志信息
logging:
  level:
    org.springframework.cloud.openfeign: debug
    org.springframework.cloud.sleuth: debug
```



### 演示接口完善

以下接口只是为了演示造的数据，并没有整合DB。

sleuth-order9033查询订单详情的接口，如下图：

![](https://img.java-family.cn/sleuth/3.png)

sleuth-product9032的查询商品详情的接口，如下图：

![](https://img.java-family.cn/sleuth/4.png)



gateway-sleuth9031网关路由配置如下：

![](https://img.java-family.cn/sleuth/5.png)



### 测试

启动上述三个服务，浏览器直接访问：http://localhost:9031/order/get/12

观察控制台日志输出，如下图：

![](https://img.java-family.cn/sleuth/6.png)

日志格式中总共有四个参数，含义分别如下：

- 第一个：服务名称
- 第二个：traceId，唯一标识一条链路
- 第三个：spanId，链路中的基本工作单元id
- 第四个：表示是否将数据输出到其他服务，true则会把信息输出到其他可视化的服务上观察，这里并未整合zipkin，所以是false

好了，至此整合完成了，不禁心里倒吸一口凉气，直接看日志那不是眼睛要看瞎了..........

> 案例源码已经上传，公众号【码猿技术专栏】回复关键词 **9528**获取。



## 什么是ZipKin？

Zipkin 是 Twitter 的一个开源项目，它基于Google Dapper实现，它致力于收集服务的定时数据，

以解决微服务架构中的延迟问题，包括数据的**收集、存储、查找和展现**。

ZipKin的基础架构如下图：

![](https://img.java-family.cn/sleuth/7.png)

Zipkin共分为4个核心的组件，如下：

- **Collector**：收集器组件，它主要用于处理从外部系统发送过来的跟踪信息，将这些信息转换为Zipkin内部处理的 Span 格式，以支持后续的存储、分析、展示等功能。

- **Storage**：存储组件，它主要对处理收集器接收到的跟踪信息，默认会将这些信息存储在内存中，我们也可以修改此存储策略，通过使用其他存储组件将跟踪信息存储到数据库中

- **RESTful API**：API 组件，它主要用来提供外部访问接口。比如给客户端展示跟踪信息，或是外接系统访问以实现监控等。

- **UI**：基于API组件实现的上层应用。通过UI组件用户可以方便而有直观地查询和分析跟踪信息

zipkin分为服务端和客户端，服务端主要用来收集跟踪数据并且展示，客户端主要功能是发送给服务端，微服务的应用也就是客户端，这样一旦发生调用，就会触发监听器将sleuth日志数据传输给服务端。

## zipkin服务端如何搭建？

首先需要下载服务端的jar包，地址：https://search.maven.org/artifact/io.zipkin/zipkin-server/2.23.4/jar

下载完成将会得到一个jar包，如下图：

![](https://img.java-family.cn/sleuth/8.png)

直接启动这个jar，命令如下：

```shell
java -jar zipkin-server-2.23.4-exec.jar
```

出现以下界面表示启动完成：

![](https://img.java-family.cn/sleuth/9.png)

此时可以访问zipkin的UI界面，地址：http://localhost:9411，界面如下：

![](https://img.java-family.cn/sleuth/10.png)

以上是通过下载jar的方式搭建服务端，当然也有其他方式安装，比如docker，自己去尝试一下吧，陈某就不再演示了。



## zipKin客户端如何搭建？

服务端只是跟踪数据的收集和展示，客户端才是生成和传输数据的一端，下面详细介绍一下如何搭建一个客户端。

还是上述例子的三个微服务，直接添加zipkin的依赖，如下：

```xml
<!--链路追踪 zipkin依赖，其中包含Sleuth的依赖-->
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

**注意**：由于`spring-cloud-starter-zipkin`中已经包含了Spring Cloud Sleuth依赖，因此只需要引入上述一个依赖即可。



配置文件需要配置一下zipkin服务端的地址，配置如下：

```yaml
spring:
  cloud:
  sleuth:
    sampler:
      # 日志数据采样百分比，默认0.1(10%)，这里为了测试设置成了100%，生产环境只需要0.1即可
      probability: 1.0
  zipkin:
      #zipkin server的请求地址
    base-url: http://127.0.0.1:9411
      #让nacos把它当成一个URL，而不要当做服务名
    discovery-client-enabled: false
```

上述配置完成后启动服务即可，此时访问：http://localhost:9031/order/get/12

调用接口之后，再次访问zipkin的UI界面，如下图：

![](https://img.java-family.cn/sleuth/11.png)

可以看到刚才调用的接口已经被监控到了，点击`SHOW`进入详情查看，如下图：

![](https://img.java-family.cn/sleuth/12.png)

可以看到**左边**展示了一条完整的链路，包括服务名称、耗时，**右边**展示服务调用的相关信息，包括开始、结束时间、请求url，请求方式.....

除了调用链路的相关信息，还可以清楚看到每个服务的依赖如下图，如下图：

![](https://img.java-family.cn/sleuth/13.png)





## zipKin的数据传输方式如何切换？

zipkin默认的传输方式是HTTP，但是这里存在一个问题，一旦传输过程中客户端和服务端断掉了，那么这条跟踪日志信息将会丢失。

当然zipkin还支持**MQ**方式的传输，支持消息中间件有如下几种：

- ActiveMQ
- RabbitMQ
- Kafka

使用MQ方式传输不仅能够保证消息丢失的问题，还能提高传输效率，**生产中推荐MQ传输方式**。

**那么问题来了，如何切换呢？**

其实方式很简单，下面陈某以RabbitMQ为例介绍一下。

### 1、服务端连接RabbitMQ

运行服务端并且连接RabbitMQ，命令如下：

```shell
java -jar zipkin-server-2.23.4-exec.jar --zipkin.collector.rabbitmq.addresses=localhost --zipkin.collector.rabbitmq.username=guest --zipkin.collector.rabbitmq.password=guest
```

命令分析如下：

- `zipkin.collector.rabbitmq.addresses`：MQ地址
- `zipkin.collector.rabbitmq.username`：用户名
- `zipkin.collector.rabbitmq.password`：密码



### 2、客户端添加RabbitMQ

既然使用MQ传输，肯定是要添加对应的依赖和配置了，添加RabbitMQ依赖如下：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

配置MQ的地址、用户名、密码，配置如下：

```yaml
spring:
  rabbitmq:
    addresses: 127.0.0.1
    username: guest
    password: guest
```



### 3、配置文件中传输方式切换

`spring.cloud.zipkin.sender.type`这个配置就是用来切换传输方式的，取值为`rabbit`则表示使用rabbitMQ进行数据传输。

配置如下：

```yaml
spring:
  cloud:
  zipkin:
    sender:
     ## 使用rabbitMQ进行数据传输
      type: rabbit
```

注意：使用MQ传输，则`spring.cloud.zipkin.sender.base-url`可以去掉。

完整的配置如下图：

![](https://img.java-family.cn/sleuth/14.png)

### 4、测试

既然使用MQ传输，那么我们不启动服务端也是能够成功传输的，浏览器访问：http://localhost:9031/order/get/12

此时发现服务并没有报异常，在看RabbitMQ中已经有数据传输过来了，存在`zipkin`这个队列中，如下图：

![](https://img.java-family.cn/sleuth/15.png)

可以看到有消息未被消费，点进去可以看到消息内容就是Trace、Span相关信息。

好了，我们启动服务端，命令如下：

```shell
java -jar zipkin-server-2.23.4-exec.jar --zipkin.collector.rabbitmq.addresses=localhost --zipkin.collector.rabbitmq.username=guest --zipkin.collector.rabbitmq.password=guest
```

服务端启动后发现zipkin队列中的消息瞬间被消费了，查看zipkin的UI界面发现已经生成了链路信息，如下图：

![](https://img.java-family.cn/sleuth/16.png)





## zipkin如何持久化？

zipkin的信息默认是存储在内存中，服务端一旦重启信息将会丢失，但是zipkin提供了可插拔式的存储。

zipkin支持以下四种存储方式：

- 内存：服务重启将会失效，不推荐
- MySQL：数据量越大性能较低
- Elasticsearch：主流的解决方案，推荐使用
- Cassandra：技术太牛批，用的人少，自己选择，不过官方推荐

今天陈某就以MySQL为例介绍一下zipkin如何持久化，Elasticsearch放在下一篇，篇幅有点长。

### 1、创建数据库

zipkin服务端的MySQL建表SQL在源码中的`zipkin-storage/mysql-v1/src/main/resources/mysql.sql`中，这份SQL文件我会放在案例源码中。

> github地址：https://github.com/openzipkin/zipkin/blob/master/zipkin-storage/mysql-v1/src/main/resources/mysql.sql

创建的数据库：**zipkin**（名称任意），导入建表SQL，新建的数据库表如下图：

![](https://img.java-family.cn/sleuth/17.png)

### 2、服务端配置MySQL

服务端配置很简单，运行如下命令：

```shell
java -jar zipkin-server-2.23.4-exec.jar --STORAGE_TYPE=mysql --MYSQL_HOST=127.0.0.1 --MYSQL_TCP_PORT=3306 --MYSQL_DB=zipkin --MYSQL_USER=root --MYSQL_PASS=Nov2014
```

上述命令参数分析如下：

- **STORAGE_TYPE**：指定存储的方式，默认内存形式
- **MYSQL_HOST**：MySQL的ip地址，默认localhost
- **MYSQL_TCP_PORT**：MySQL的端口号，默认端口3306
- **MYSQL_DB**：MySQL中的数据库名称，默认是zipkin
- **MYSQL_USER**：用户名
- **MYSQL_PASS**：密码

陈某是如何记得这些参数的？废话，肯定记不住，随时查看下源码不就得了，这些配置都在源码的`/zipkin-server/src/main/resources/zipkin-server-shared.yml`这个配置文件中，比如上述MySQL的相关配置，如下图：

![](https://img.java-family.cn/sleuth/18.png)

zipkin服务端的所有配置项都在这里，没事去翻翻看。

> github地址：https://github.com/openzipkin/zipkin/blob/master/zipkin-server/src/main/resources/zipkin-server-shared.yml

那么采用rabbitMQ传输方式、MySQL持久化方式，完整的命令如下：

```shell
java -jar zipkin-server-2.23.4-exec.jar --STORAGE_TYPE=mysql --MYSQL_HOST=127.0.0.1 --MYSQL_TCP_PORT=3306 --MYSQL_DB=zipkin --MYSQL_USER=root --MYSQL_PASS=Nov2014 --zipkin.collector.rabbitmq.addresses=localhost --zipkin.collector.rabbitmq.username=guest --zipkin.collector.rabbitmq.password=guest
```

持久化是服务端做的事，和客户端无关，因此到这就完事了，陈某就不再测试了，自己动手试试吧。



## 总结

前面介绍了这么多，不知道大家有没有仔细看，陈某总结一下吧：

- Spring Cloud Sleuth 作为链路追踪的一种组件，只提供了日志采集，日志打印的功能，并没有可视化的UI界面
- zipkin提供了强大的日志追踪分析、可视化、服务依赖分析等相关功能，结合Spring Cloud Sleuth作为一种主流的解决方案
- zipkin生产环境建议切换的MQ传输模式，这样做有两个优点
  - 防止数据丢失
  - MQ异步解耦，性能提升很大
- zipkin默认是内存的形式存储，MySQL虽然也是一种方式，但是随着数据量越大，性能越差，因此生产环境建议采用Elasticsearch，下一篇文章介绍。

> 案例源码已经上传，公众号【码猿技术专栏】回复关键词**9528**获取。

## 最后说一句（求关注，别白嫖我）

陈某每一篇原创文章都是精心输出，尤其是《Spring Cloud 进阶》专栏的文章，知识点太多，要想讲的细，必须要花很多时间准备，从知识点到源码demo。

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！



































