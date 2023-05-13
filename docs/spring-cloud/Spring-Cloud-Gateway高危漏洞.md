**大家好，我是不才陈某~**

Log4j2 的核弹级漏洞刚告一段落，**Spring Cloud Gateway** 又突发高危漏洞，又得折腾了。。。

2022年3月1日，Spring官方发布了关于Spring Cloud Gateway的两个CVE漏洞，分别为**CVE-2022-22946**与**CVE-2022-22947**：

**版本/分支/tag： 3.4.X**
**问题描述**：

**![](https://mmbiz.qpic.cn/mmbiz_png/AokYxdF6ctt1NeiayJzDScfIZZrNMaNAkIgMQMKP9OLVWa4hv9WFfXnfYjMLy4WAib31FW8cXMZ2BVSoYuYAia7EQ/640?wx_fmt=png)**





Spring Cloud Gateway 是 Spring Cloud 下的一个项目，该项目是基于 Spring 5.0、Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效、统一的 API 路由管理方式。



## **漏洞1**：Spring Cloud Gateway 远程代码执行漏洞（CVE-2022-22947）

3 月 1 日，VMware 官方发布安全公告，声明对 Spring Cloud Gateway 中的一处命令注入漏洞进行了修复，漏洞编号为 CVE-2022-22947：

https://tanzu.vmware.com/security/cve-2022-22947



**漏洞描述**

使用 Spring Cloud Gateway 的应用如果对外暴露了 Gateway Actuator 端点时，则可能存在被 CVE-2022-22947 漏洞利用的风险。攻击者可通过利用此漏洞执行 SpEL 表达式，允许在远程主机上进行任意远程执行。，获取系统权限。

**影响范围**



漏洞利用的前置条件：

1. 除了 Spring Cloud Gateway 外，程序还用到了 Spring Boot Actuator 组件（它用于对外提供 /actuator/ 接口）；
2. Spring 配置对外暴露 gateway 接口，如 application.properties 配置为：

```properties
# 默认为true
management.endpoint.gateway.enabled=true
# 以逗号分隔的一系列值，默认为 health# 若包含 gateway 即表示对外提供 Spring Cloud Gateway 接口
management.endpoints.web.exposure.include=gateway
```



漏洞影响的 Spring Cloud Gateway 版本范围：

- Spring Cloud Gateway 3.1.x < 3.1.1
- Spring Cloud Gateway 3.0.x < 3.0.7
- 其他旧的、不受支持的 Spring Cloud Gateway 版本



**解决方案**

更新升级 Spring Cloud Gateway 到以下安全版本：

- Spring Cloud Gateway 3.1.1
- Spring Cloud Gateway 3.0.7

或者在不考虑影响业务的情况下禁用 Gateway actuator 接口：

在application.properties 中设置 ：

```properties
management.endpoint.gateway.enabled 为 false。
```



## **漏洞2**：CVE-2022-22946：Spring Cloud Gateway HTTP2 不安全的 TrustManager



**漏洞描述**

使用配置为启用 HTTP2 且未设置密钥存储或受信任证书的 Spring Cloud Gateway 的应用程序将被配置为使用不安全的 TrustManager。这使得网关能够使用无效或自定义证书连接到远程服务。

**影响范围**

Spring Cloud Gateway = 3.1.0

**解决方案**

官方已经发布安全版本，请升级到3.1.1+

**参考资料**

官方已经发布安全版本，请升级到3.1.1+

- https://tanzu.vmware.com/security/cve-2022-22946
- https://tanzu.vmware.com/security/cve-2022-22947
