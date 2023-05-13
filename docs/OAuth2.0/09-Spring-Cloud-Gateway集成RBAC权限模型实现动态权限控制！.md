
这篇文章介绍下网关层如何集成RBAC权限模型进行认证鉴权，文章目录如下：

![](https://img.java-family.cn/Spring%20Security/213.png)

## 什么是RBAC权限模型？

**RBAC**(Role-Based Access Control)**基于角色访问控制**，目前使用最为广泛的权限模型。

相信大家对这种权限模型已经比较了解了。此模型有三个**用户**、**角色**和**权限**，在传统的权限模型用户直接关联加了角色层，解耦了用户和权限，使得权限系统有了更清晰的职责划分和更高的灵活度。

![](https://img.java-family.cn/Spring%20Security/205.png)

以上五张表的SQL就不再详细贴出来了，都会放在案例源码的**doc**目录下，如下图：

![](https://img.java-family.cn/Spring%20Security/212.png)



## 设计思路

**RBAC**权限模型是基于角色的，因此在Spring Security中的权限就是角色，具体的认证授权流程如下：

1. 用户登录申请令牌
2. 通过**UserDetailService**查询、加载用户信息、比如密码、权限(角色)....封装到**UserDetails**中
2. 令牌申请成功，携带令牌访问资源
2. 网关层面比较访问的**URL所需要的权限**（Redis中）是否与当前令牌具备的权限有交集。有交集则表示具备访问该URL的权限。
2. 具备权限则访问，否则拒绝

上述只是大致的流程，其中还有一些细节有待商榷，如下：

### **1、URL对应的权限如何维护？**

这个就比较容易实现了，涉及到**RBAC**权限模式的三张表，分别为**权限表**、**角色表**、**权限角色**对应关系表。具体实现流程如下：

1. 项目启动时将权限（URL）和角色的对应关系加载到Redis中。
2. 对于管理界面涉及到URL相应关系的变动要实时的变更到Redis。

比如权限中有这么一条数据，如下：

![](https://img.java-family.cn/Spring%20Security/206.png)

其中的 **/order/info** 这个URL就是一个权限，管理员可以对其分配给指定的角色。

### **2、如何实现Restful风格的权限控制？**

restful风格的接口URL是相同的，不同的只是请求方式，因此要想做到权限的精细控制还需要保留请求方式，比如POST，GET，PUT，DELETE....

可以在权限表中的url字段放置一个**method**标识，比如POST，此时的完整URL为：**POST:/order/info**

> 当然`*:/order/info`中的星号表示一切请求方式都满足。



### **3、这样能实现动态权限控制吗？**

权限的控制方式有很多种，比如Security自身的注解、方法拦截，其实扩展Spring Security也是可以实现动态权限控制的，这个在后面的文章中会单独介绍！

陈某此篇文章是将权限、角色对应关系存入Redis中，因此想要实现动态权限控制只需要在Redis中维护这种关系即可。Redis中的数据如下：

![](https://img.java-family.cn/Spring%20Security/207.png)



## 案例实现

此篇文章还是基于以下三个模块进行改动，有不清楚的可以查看陈某往期文章。

| 名称                     | 功能               |
| ------------------------ | ------------------ |
| oauth2-cloud-auth-server | OAuth2.0认证授权服 |
| oauth2-cloud-gateway     | 网关服务           |
| oauth2-cloud-auth-common | 公共模块           |

涉及到的更改目录如下图：

![](https://img.java-family.cn/Spring%20Security/211.png)

### **1、从数据库加载URL<->角色对应关系到Redis**

在项目启动之初直接读取数据库中的权限加载到Redis中，当然方法有很多种，自己根据情况选择。代码如下：

![](https://img.java-family.cn/Spring%20Security/208.png)

此处代码在**oauth2-cloud-auth-server**模块下。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

### **2、实现UserDetailsService加载权限**

**UserDetailsService**相信大家都已经很熟悉了，主要作用就是根据用户名从数据库中加载用户的详细信息。

代码如下：

![](https://img.java-family.cn/Spring%20Security/209.png)

**①**处的代码是将通过JPA从数据库中查询用户信息并且组装角色，必须是以 **ROLE_** 开头。

**②**处的代码是将获取的角色封装进入**authorities**向下传递。

此处代码在**oauth2-cloud-auth-server**模块下。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

### **3、鉴权管理器中校验权限**

在上篇文章中[实战干货！Spring Cloud Gateway 整合 OAuth2.0 实现分布式统一认证授权！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247503249&amp;idx=1&amp;sn=b33ae3ff70a08b17ee0779d6ccb30b53&amp;chksm=fcf7125ccb809b4aa4985da09e620e06c606754e6a72681c93dcc88bdc9aa7ba0cb64f52dbc3&token=284256295&lang=zh_CN#rd)详细介绍了鉴权管理器的作用，这里就不再细说了。代码如下：

![](https://img.java-family.cn/Spring%20Security/210.png)

**①**处的代码是将请求URL组装成restful风格的，比如**POST:/order/info**

**②**处的代码是从Redis中取出URL和角色对应关系遍历，通过**AntPathMatcher**进行比对，获取当前请求URL的所需的角色。

**③**处的代码就是比较当前URL所需的角色和当前用户的角色，分为两步：

- 如果是超级管理员，则直接放行，不必比较权限
- 不是超级管理员就需要比较角色，有交集才能放行

此处的代码在**oauth2-cloud-gateway**模块中。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

### **4、总结**

关键代码就是上述三处，另外关于一些**DAO**层的相关代码就不再贴出来了，自己下载源码看看！

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！



## 附加的更改

这篇文章中顺带将客户端信息也放在了数据库中，前面的文章都是放在内存中。

数据库中新建一张表，SQL如下：

```sql
CREATE TABLE `oauth_client_details` (
  `client_id` varchar(48) NOT NULL COMMENT '客户端id',
  `resource_ids` varchar(256) DEFAULT NULL COMMENT '资源的id，多个用逗号分隔',
  `client_secret` varchar(256) DEFAULT NULL COMMENT '客户端的秘钥',
  `scope` varchar(256) DEFAULT NULL COMMENT '客户端的权限，多个用逗号分隔',
  `authorized_grant_types` varchar(256) DEFAULT NULL COMMENT '授权类型，五种，多个用逗号分隔',
  `web_server_redirect_uri` varchar(256) DEFAULT NULL COMMENT '授权码模式的跳转uri',
  `authorities` varchar(256) DEFAULT NULL COMMENT '权限，多个用逗号分隔',
  `access_token_validity` int(11) DEFAULT NULL COMMENT 'access_token的过期时间，单位毫秒，覆盖掉硬编码',
  `refresh_token_validity` int(11) DEFAULT NULL COMMENT 'refresh_token的过期时间，单位毫秒，覆盖掉硬编码',
  `additional_information` varchar(4096) DEFAULT NULL COMMENT '扩展字段，JSON',
  `autoapprove` varchar(256) DEFAULT NULL COMMENT '默认false，是否自动授权',
  PRIMARY KEY (`client_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

认证服务中的OAuth2.0的配置文件中将客户端的信息从数据库中加载，该实现类为**JdbcClientDetailsService**，关键代码如下：

```java
	@Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        //使用JdbcClientDetailsService，从数据库中加载客户端的信息
        clients.withClientDetails(new JdbcClientDetailsService(dataSource));
    }
```

## 总结

本篇文章介绍了网关集成RBAC权限模型进行认证鉴权，核心思想就是将权限信息加载Redis缓存中，在网关层面的鉴权管理器中进行权限的校验，其中还整合了Restful风格的URL。

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！

















