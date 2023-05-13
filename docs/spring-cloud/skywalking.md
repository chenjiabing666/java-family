
# 分布式链路追踪组件 Skywalking
本篇文章介绍链路追踪的另外一种解决方案Skywalking，文章目录如下：

![](https://img.java-family.cn/skywalking/34.png)

## 什么是Skywalking？

上一篇文章介绍了分布式链路追踪的一种方式：**Spring Cloud Sleuth+ZipKin**，这种方案目前也是有很多企业在用，但是作为程序员要的追逐一些新奇的技术，Skywalking作为后起之秀也是值得大家去学习的。

skywalking是一个优秀的**国产开源框架**，**2015**年由个人吴晟（华为开发者）开源 ， **2017**年加入Apache孵化器。短短两年就被Apache收入麾下，实力可见一斑。

skywalking支持dubbo，SpringCloud，SpringBoot集成，**代码无侵入，通信方式采用GRPC，性能较好，实现方式是java探针，支持告警，支持JVM监控，支持全局调用统计**等等，功能较完善。

## Skywalking和Spring Cloud Sleuth+ZipKin如何选型？

Skywalking相比于zipkin还是有很大的优势的，如下：

- skywalking采用字节码增强的技术实现代码无侵入，zipKin代码侵入性比较高
- skywalking功能比较丰富，报表统计，UI界面更加人性化

**个人建议**：如果是新的架构，建议优先选择skywalking。



## Skywalking架构是怎样的？

skywalking和zipkin一样，也分为服务端和客户端，服务端负责收集日志数据并且展示，架构如下：

![](https://img.java-family.cn/skywalking/1.png)

上述架构图中主要分为四个部分，如下：

- **上面的Agent**：负责收集日志数据，并且传递给中间的OAP服务器
- **中间的OAP**：负责接收 Agent 发送的 Tracing 和Metric的数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。
- **左面的UI**：负责提供web控制台，查看链路，查看各种指标，性能等等。
- **右面Storage**：负责数据的存储，支持多种存储类型。

看了架构图之后，思路很清晰了，Agent负责收集日志传输数据，通过GRPC的方式传递给OAP进行分析并且存储到数据库中，最终通过UI界面将分析的统计报表、服务依赖、拓扑关系图展示出来。



## 服务端如何搭建？

skywalking同样是通过jar包方式启动，需要下载jar包，地址：https://skywalking.apache.org/downloads/

### 1、下载安装包

选择**V8.7.0**这个版本，如下图：

![](https://img.java-family.cn/skywalking/2.png)

> 以上只是陈某的选择，可以按照自己的需要选择其他版本



解压之后完整目录如下图：

![](https://img.java-family.cn/skywalking/3.png)

重要的目录结构分析如下：

- **agent**：客户端需要指定的目录，其中有一个jar，就是负责和客户端整合收集日志
- **bin**：服务端启动的脚本
- **config**：一些配置文件的目录
- **logs**：oap服务的日志目录
- **oap-libs**：oap所需的依赖目录
- **webapp**：UI服务的目录

### 2、配置修改

启动之前需要对配置文件做一些修改，修改如下：

**1、/config/application.yml**

这个是oap服务的配置文件，需要修改注册中心为nacos，如下图：

![](https://img.java-family.cn/skywalking/4.png)

**配置①**：修改默认注册中心选择nacos，这样就不用在启动参数中指定了。

**配置②**：修改nacos的相关配置，由于陈某是本地的，则不用改动，根据自己情况修改。



**2、webapp/webapp.yml**

这个是UI服务的配置文件，其中有一个**server.port**配置，是UI服务的端口，默认**8080**，陈某将其改成**8888**，避免端口冲突，如下图：

![](https://img.java-family.cn/skywalking/5.png)



### 3、启动服务

启动命令在/bin目录下，这里需要启动两个服务，如下：

- **oap服务**：对应的启动脚本oapService.bat，Linux下对应的后缀是sh
- **UI服务**：对应的启动脚本webappService.bat，Linux下对应的后缀是sh

当然还有一个**startup.bat**启动文件，可以直接启动上述两个服务，我们可以直接使用这个脚本，直接双击，将会弹出两个窗口则表示启动成功，如下图：

![](https://img.java-family.cn/skywalking/6.png)



此时直接访问：http://localhost:8888/，直接进入UI端，如下图：

![](https://img.java-family.cn/skywalking/7.png)

## 客户端如何搭建？

客户端也就是单个微服务，由于Skywalking采用字节码增强技术，因此对于微服务无代码侵入，只要是普通的微服务即可，不需要引入什么依赖。

还是上一篇Spring Cloud Sleuth的三个服务，如下：

- **skywalking-product1001**：商品微服务
- **skywalking-order1002**：订单微服务
- **skywalking-gateway1003**：网关微服务

> 案例源码已经上传，关注公众号：码猿技术专栏，回复关键词 **9528**获取！

案例源码结构目录如下图：

![](https://img.java-family.cn/skywalking/8.png)



想要传输数据必须借助skywalking提供的agent，只需要在启动参数指定即可，命令如下：

```shell
-javaagent:E:\springcloud\apache-skywalking-apm-es7-8.7.0\apache-skywalking-apm-bin-es7\agent\skywalking-agent.jar
-Dskywalking.agent.service_name=skywalking-product-service
-Dskywalking.collector.backend_service=127.0.0.1:11800
```

上述命令解析如下：

- **-javaagent**：指定skywalking中的agent中的`skywalking-agent.jar`的路径
- **-Dskywalking.agent.service_name**：指定在skywalking中的服务名称，一般是微服务的`spring.application.name`
- **-Dskywalking.collector.backend_service**：指定oap服务绑定的地址，由于陈某这里是本地，并且oap服务默认的端口是11800，因此只需要配置为`127.0.0.1:11800`

上述参数可以在命令行通过`java -jar xxx`指定，在IDEA中操作如下图：

![](https://img.java-family.cn/skywalking/9.png)

上述三个微服务都需要配置skywalking的启动配置，配置成功后正常启动即可。

> 注意：agent的jar包路径不能包含中文，不能有空格，否则运行不成功。

成功启动后，直接通过网关访问：http://localhost:1003/order/get/12，返回如下信息：

![](https://img.java-family.cn/skywalking/10.png)

此时查看skywalking的UI端，可以看到三个服务已经监控成功了，如下图：

![](https://img.java-family.cn/skywalking/11.png)

服务之前的依赖关系也是可以很清楚的看到，如下图：

![](https://img.java-family.cn/skywalking/12.png)

请求链路的信息也是能够很清楚的看到，比如请求的url，执行时间、调用的服务，如下图：

![](https://img.java-family.cn/skywalking/13.png)

感觉怎样？是不是很高端，比zipkin的功能更加丰富。

## 数据如何持久化？

你会发现只要服务端重启之后，这些链路追踪数据将会丢失了，因为skywalking默认持久化的方式是存储在内存中。

当然这里也是可以通过插拔方式的替换掉存储中间件，企业中往往是使用**ES**存储，陈某这章介绍一下**MySQL**的方式存储，ES方式后面单独开一篇文章介绍。

### 1、修改配置文件

修改config/application.yml文件中的存储方式，总共需要修改两处地方。

- 修改默认的存储方式为mysql，如下图：

![](https://img.java-family.cn/skywalking/14.png)

- 修改Mysql相关的信息，比如用户名、密码等，如下图：

![](https://img.java-family.cn/skywalking/15.png)



### 2、添加MySQL的jdbc依赖

默认的oap中是没有jdbc驱动依赖，因此需要我们手动添加一下，只需要将驱动的jar放在**oap-libs**文件夹中，如下图：

![](https://img.java-family.cn/skywalking/16.png)

好了，已经配置完成，启动服务端，在skywalking这个数据库中将会自动创建表，如下图：

![](https://img.java-family.cn/skywalking/17.png)

陈某这里就不再测试了，自己耍耍吧.......

## 日志监控如何做？

在skywalking的UI端有一个日志的模块，用于收集客户端的日志，默认是没有数据的，那么需要如何将日志数据传输到skywalking中呢？

日志框架的种类很多，比较出名的有log4j，logback，log4j2，陈某就以**logback**为例子介绍一下如何配置，官方文档如下：

- log4j：https://skywalking.apache.org/docs/skywalking-java/v8.8.0/en/setup/service-agent/java-agent/application-toolkit-log4j-1.x/
- log4j2：https://skywalking.apache.org/docs/skywalking-java/v8.8.0/en/setup/service-agent/java-agent/application-toolkit-log4j-2.x/
- logback：https://skywalking.apache.org/docs/skywalking-java/v8.8.0/en/setup/service-agent/java-agent/application-toolkit-logback-1.x/

### 1、添加依赖

根据官方文档，需要先添加依赖，如下：

```xml
<dependency>
   <groupId>org.apache.skywalking</groupId>
   <artifactId>apm-toolkit-logback-1.x</artifactId>
   <version>${project.release.version}</version>
</dependency>
```

### 2、添加配置文件

新建一个`logback-spring.xml`放在resource目录下，配置如下图：

![](https://img.java-family.cn/skywalking/18.png)

以上配置全部都是拷贝官方文档的，对于日志配置有不理解的地方，可以看陈某之前的文章：[Spring Boot第三弹，一文带你搞懂日志如何配置？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247484716&amp;idx=1&amp;sn=04db24b54f73e7eaf72e393cc1f215a3&amp;chksm=fcf4dae1cb8353f738b3b61d9a935e256f2f885d13dff5d6e8cff03b85a5153d4aee8fc8328e&token=1889058377&lang=zh_CN#rd)

配置完成之后，启动服务，访问：http://localhost:1003/order/get/12。

控制台打印的日志如下图：

![](https://img.java-family.cn/skywalking/19.png)

可以看到已经打印出了traceId。

skywalking中的日志模块输出的日志如下图：

![](https://img.java-family.cn/skywalking/20.png)

日志已经传输到了skywalking中..............

**注意**：如果**agent**和**oap**服务不在同一台服务器上，需要在`/agent/config/agent.config`配置文件末尾添加如下配置：

```properties
plugin.toolkit.log.grpc.reporter.server_host=${SW_GRPC_LOG_SERVER_HOST:10.10.10.1}
plugin.toolkit.log.grpc.reporter.server_port=${SW_GRPC_LOG_SERVER_PORT:11800}
plugin.toolkit.log.grpc.reporter.max_message_size=${SW_GRPC_LOG_MAX_MESSAGE_SIZE:10485760}
plugin.toolkit.log.grpc.reporter.upstream_timeout=${SW_GRPC_LOG_GRPC_UPSTREAM_TIMEOUT:30}
```

配置分析如下图：

![](https://img.java-family.cn/skywalking/21.png)



## 性能剖析如何做？

skywalking在性能剖析方面真的是非常强大，提供到基于堆栈的分析结果，能够让运维人员一眼定位到问题。

新建一个`/order/list`接口，如下：

![](https://img.java-family.cn/skywalking/22.png)

上述代码中休眠了2秒，看看如何在skywalking中定位这个问题。

在性能剖析模块->新建任务->选择服务、填写端点、监控时间，操作如下图：

![](https://img.java-family.cn/skywalking/23.png)

上图中选择了最大采样数为5，则直接访问5次：http://localhost:1003/order/list，然后选择这个任务将会出现监控到的数据，如下图：

![](https://img.java-family.cn/skywalking/24.png)

上图中可以看到` {GET}/order/list`这个接口上耗费了2秒以上，因此选择这个接口点击分析，可以看到详细的堆栈信息，如下图：

![](https://img.java-family.cn/skywalking/25.png)

如何定位到睡眠2秒钟的那一行代码呢？直接往下翻，如下图：

![](https://img.java-family.cn/skywalking/26.png)

是不是很清楚了，在OrderController这个接口线程睡眠了两秒........



## 监控告警如何做？

对于服务的异常信息，比如接口有较长延迟，skywalking也做出了告警功能，如下图：

![](https://img.java-family.cn/skywalking/27.png)

skywalking中有一些默认的告警规则，如下：

- 最近3分钟内服务的平均响应时间超过1秒
- 最近2分钟服务成功率低于80%
- 最近3分钟90%服务响应时间超过1秒
- 最近2分钟内服务实例的平均响应时间超过1秒

当然除了以上四种，随着Skywalking不断迭代也会新增其他规则，这些规则的配置在`config/alarm-settings.yml`配置文件中，如下图：

![](https://img.java-family.cn/skywalking/28.png)

每个规则都由相同的属性组成，这些属性的含义如下图：

![](https://img.java-family.cn/skywalking/29.png)

如果想要调整默认的规则，比如监控返回的信息，监控的参数等等，只需要改动上述配置文件中的参数即可。

当然除了以上默认的几种规则，skywalking还适配了一些钩子（**webhooks**）。其实就是相当于一个回调，一旦触发了上述规则告警，skywalking则会调用配置的webhook，这样开发者就可以定制一些处理方法，比如发送**邮件**、**微信**、**钉钉**通知运维人员处理。

当然这个钩子也是有些规则的，如下：

- POST请求
- **application/json** 接收数据
- 接收的参数必须是AlarmMessage中指定的参数。

**注意**：AlarmMessage这个类随着skywalking版本的迭代可能出现不同，一定要到对应版本源码中去找到这个类，拷贝其中的属性。这个类在源码的路径：`org.apache.skywalking.oap.server.core.alarm`，如下图：

![](https://img.java-family.cn/skywalking/30.png)

新建一个告警模块：**skywalking-alarm1004**，其中利用webhook定义一个接口，如下：

![](https://img.java-family.cn/skywalking/31.png)

接口定制完成后，只需要在`config/alarm-settings.yml`配置文件中添加这个钩子，如下图：

![](https://img.java-family.cn/skywalking/32.png)

好了，这就已经配置完成了，测试也很简单，还是调用上面案例中的睡眠两秒的接口：http://localhost:1003/order/list，多调用几次，则会触发告警，控制台打印日志如下：

![](https://img.java-family.cn/skywalking/33.png)



## 总结

本篇文章介绍了链路追踪解决方案Skywalking，主要讲解了服务端搭建、客户端搭建、数据持久化、日志监控、性能剖析这几个知识点，每个知识点都非常重要。

> 案例源码已经上传，关注公众号：码猿技术专栏，回复关键词 **9528**获取！

## 最后说一句（求关注，别白嫖我）

陈某每一篇原创文章都是精心输出，尤其是《Spring Cloud 进阶》专栏的文章，知识点太多，要想讲的细，必须要花很多时间准备，从知识点到源码demo。

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！















