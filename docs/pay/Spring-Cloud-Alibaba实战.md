**大家好，我是不才陈某~**

前期一直在更新**《Spring Cloud 进阶》**这个专栏，很多读者反馈知识太杂了，想要一时间掌握非常困难，想要陈某出一个实战项目，这样能够更加深入的理解。

陈某趁着隔离和徒弟两人熬了一个月时间出了一个Spring Cloud Alibaba微服务的实战项目，其中运用了当下主流的微服务技术，比如Nacos、Sentinel、Seata、OpenFeign、Skywalking.....

项目初期整体架构图如下：

![](https://img.java-family.cn/木谷博客/1.png)

> 演示地址：http://124.221.134.51:1000

该项目运用到的主流技术：

- 引入Nacos作为注册中心、分布式配置中心，开发便捷
- 引入openFeign作为服务调用组件，贴近企业生产实际
- 引入Seata作为分布式解决方案，使得分布式事务更加简单
- 引入Sentinel作为限流熔断组件，使得微服务更加安全，通过配置再也不怕网站被爆破
- 引入Skywalking作为分布式链路追踪组件，代码无侵入，使得异常分析，链路定位更加简单
- 引入RBAC权限模型，灵活的权限控制，按钮级别的细粒度权限控制，满足绝大部分的权限需求
- 引入Spring Security作为认证授权框架，完美集成OAuth2.0
- 引入ElasticSearch作为全文检索服务
- 引入Spring Boot Admin作为服务监控组件
- 引入**Swagger** 文档支持，网关层聚合API文档，不用担心文档的编写
- 引入**RabbitMQ** 消息队列，用于事件的异步拆解
- 采用**自定义参数校验注解**，轻松实现后端参数校验
- 采用 **AOP** + 自定义注解 + **Redis** 实现限制IP接口访问次数
- 使用**Docker+Docker Compose** 完全自动部署

视频已经全部录制完毕，总计**78**节课，课程大纲如下：

![](https://img.java-family.cn/木谷博客/21.png)


视频目录如下：

![](https://img.java-family.cn/木谷博客/31.png)

![](https://img.java-family.cn/木谷博客/32.png)

![](https://img.java-family.cn/木谷博客/33.png)

![](https://img.java-family.cn/木谷博客/34.png)

![](https://img.java-family.cn/木谷博客/35.png)

## 如何订阅课程？

扫描下方二维码：

![](https://img.java-family.cn/木谷博客/36.jpg)

