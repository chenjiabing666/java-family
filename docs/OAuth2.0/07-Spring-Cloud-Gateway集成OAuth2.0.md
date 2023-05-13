## 微服务认证方案

微服务认证方案目前有很多种，每个企业也是大不相同，但是总体分为两类，如下：

1. 网关只负责转发请求，认证鉴权交给每个微服务控制
2. 统一在网关层面认证鉴权，微服务只负责业务

**你们公司目前用的哪种方案？**

先来说说第一种方案，有着很大的弊端，如下：

- 代码耦合严重，每个微服务都要维护一套认证鉴权
- 无法做到统一认证鉴权，开发难度太大

第二种方案明显是比较简单的一种，优点如下：

- 实现了统一的认证鉴权，微服务只需要各司其职，专注于自身的业务
  - 代码耦合性低，方便后续的扩展


下面陈某就以第二种方案为例，整合**Spring Cloud Gateway+Spring Cloud Security** 整合出一套统一认证鉴权案例。

## 案例架构

开始撸代码之前，先来说说大致的认证鉴权流程，架构如下图：

![](https://img.java-family.cn/Spring%20Security/145.png)



**大致分为四个角色，如下**：

- **客户端**：需要访问微服务资源
- **网关**：负责转发、认证、鉴权
- **OAuth2.0授权服务**：负责认证授权颁发令牌
- **微服务集合**：提供资源的一系列服务。

**大致流程如下**：

1、客户端发出请求给网关获取令牌

2、网关收到请求，直接转发给授权服务

3、授权服务验证用户名、密码等一系列身份，通过则颁发令牌给客户端

4、客户端携带令牌请求资源，请求直接到了网关层

5、网关对令牌进行校验（**验签**、**过期时间校验**....）、鉴权（对当前令牌携带的权限）和访问资源所需的权限进行比对，如果权限有交集则通过校验，直接转发给微服务

6、微服务进行逻辑处理

针对上述架构需要新建三个服务，分别如下：

| 名称                       | 功能               |
| -------------------------- | ------------------ |
| oauth2-cloud-auth-server   | OAuth2.0认证授权服 |
| oauth2-cloud-gateway       | 网关服务           |
| oauth2-cloud-order-service | 订单资源服务       |



案例源码目录如下：

![](https://img.java-family.cn/Spring%20Security/170.png)



## 认证授权服务搭建

很多企业是将认证授权服务直接集成到网关中，这么做耦合性太高了，这里陈某直接将认证授权服务抽离出来。

认证服务的搭建这里就不再细说了，上一篇文章中已经介绍的很清楚了：[OAuth2.0实战！使用JWT令牌认证！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247502801&amp;idx=1&amp;sn=56b1af09bfa25d5e44193a7d75dfa623&amp;chksm=fcf7141ccb809d0a1b0b2d7f6d9893c7d3e560dd8996296276f0274d2578236ee87e9124810d&token=278610351&lang=zh_CN#rd)

新建一个**oauth2-cloud-auth-server**模块，目录如下：

![](https://img.java-family.cn/Spring%20Security/148.png)



和上篇文章不同的是创建了**JwtTokenUserDetailsService**这个类，用于从数据库中加载用户，如下：

![](https://img.java-family.cn/Spring%20Security/146.png)

为了演示只是模拟了从数据库中查询，其中存了两个用户，如下：

- **user**：具有ROLE_user权限
- **admin**：具有ROLE_admin、ROLE_user权限

要想这个生效，还要在security的配置文件**SecurityConfig**中指定，如下图：

![](https://img.java-family.cn/Spring%20Security/147.png)



另外还整合了注册中心Nacos，详细配置就不贴了，可以看源码。有不清楚Nacos，可以看之前文章：[五十五张图告诉你微服务的灵魂摆渡者Nacos究竟有多强？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247493854&amp;idx=1&amp;sn=4b3fb7f7e17a76000733899f511ef915&amp;chksm=fcf73713cb80be05fe4473390f946dfbaf77848d7041c30f069bcb5a3629be782f4b1121bd6a&token=278610351&lang=zh_CN#rd)

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 网关服务搭建

网关使用的是**Spring Cloud Gateway**，网关如何搭建这里就不再细说了，有不清楚的可以看之前文章：[Spring Cloud Gateway夺命连环10问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247499894&amp;idx=1&amp;sn=f1606e4c00fd15292269afe052f5bca2&amp;chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&token=278610351&lang=zh_CN#rd)

新建一个**oauth2-cloud-gateway**模块，目录如下图：

![](https://img.java-family.cn/Spring%20Security/159.png)

### **1、添加依赖**

需要添加几个OAuth2.0相关的依赖，如下：

![](https://img.java-family.cn/Spring%20Security/149.png)



### **2、JWT令牌服务配置**

使用JWT令牌，配置要和认证服务的令牌配置相同，代码如下：

![](https://img.java-family.cn/Spring%20Security/150.png)





### **3、认证管理器自定义**

新建一个**JwtAuthenticationManager**，需要实现**ReactiveAuthenticationManager**这个接口。

认证管理的作用就是获取传递过来的令牌，对其进行**解析**、**验签**、**过期时间**判定。

详细代码如下：

![](https://img.java-family.cn/Spring%20Security/151.png)

逻辑很简单，就是通过JWT令牌服务解析客户端传递的令牌，并对其进行校验，比如上传三处校验失败，抛出令牌无效的异常。

**抛出的异常如何处理？如何定制返回的结果？**

这里抛出的异常可以通过**Spring Cloud Gateway**的全局异常进行捕获，这个内容在[Spring Cloud Gateway夺命连环10问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&amp;mid=2247499894&amp;idx=1&amp;sn=f1606e4c00fd15292269afe052f5bca2&amp;chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&token=278610351&lang=zh_CN#rd)这篇文章有详细介绍。下面只贴出关键代码，如下：

![](https://img.java-family.cn/Spring%20Security/152.png)



### **4、鉴权管理器自定义**

经过认证管理器**JwtAuthenticationManager**认证成功后，就需要对令牌进行鉴权，如果该令牌无访问资源的权限，则不允通过。

新建**JwtAccessManager**，实现**ReactiveAuthorizationManager**，代码如下：

![](https://img.java-family.cn/Spring%20Security/153.png)

这里的逻辑很简单，就是**取出令牌中的权限和当前请求资源URI的权限对比**，如果有交集则通过。

**①处的代码什么意思？**

这里是直接从Redis中取出资源URI对应的权限集合，因此实际开发中需要维护**资源URI和权限的对应关系**，这里不细说，为了演示，陈某直接在项目启动的时候向Redis中添加了两个资源的权限，代码如下：

![](https://img.java-family.cn/Spring%20Security/154.png)

> 注意：实际开发中需要维护资源URI和权限的对应关系。



**②处的代码什么意思？**

这处代码就是取出令牌中的权限集合

**③处的代码什么意思？**

这处代码就是比较两者权限了，有交集，则放行。

### **5、令牌无效或者过期时定制结果**

在第4步，如果令牌失效或者过期，则会直接返回，这里需要定制提示信息。

新建一个**RequestAuthenticationEntryPoint**，实现**ServerAuthenticationEntryPoint**，代码如下：

![](https://img.java-family.cn/Spring%20Security/155.png)



### **6、无权限时定制结果**

在第4步鉴权的过程中，如果无该权限，也是会直接返回，这里也需要定制提示信息。

新建一个**RequestAccessDeniedHandler**，实现**ServerAccessDeniedHandler**，代码如下：

![](https://img.java-family.cn/Spring%20Security/156.png)



### **7、OAuth2.0相关配置**

经过上述6个步骤，相关组件已经准备就绪，现在直接配置到OAuth2.0中。

新建SecurityConfig这个配置类，标注注解 **@EnableWebFluxSecurity**，注意不是 **@EnableWebSecurity**，因为Spring Cloud Gateway是基于Flux实现的。详细代码如下：

![](https://img.java-family.cn/Spring%20Security/157.png)

需要配置的内容如下：

- 认证过滤器，其中利用了认证管理器对令牌的校验
- 鉴权管理器、令牌失效异常处理、无权限访问异常处理
- 白名单配置
- 跨域过滤器的配置

### **8、全局过滤器定制**

试想一下：**网关层面认证鉴权成功后，下游微服务如何获取到当前用户的详细信息？**

陈某这里是将令牌携带的用户信息解析出来，封装成**JSON**数据，然后通过**Base64**加密，放入到请求头中，转发给下游微服务。

这样一来，下游微服务只需要解密请求头中的JSON数据，即可获取用户的详细信息。

因此需要在网关中定义一个全局过滤器，用来拦截请求，解析令牌，关键代码如下：

![](https://img.java-family.cn/Spring%20Security/158.png)

上述代码逻辑如下：

- 检查是否是白名单，白名单直接放行
- 检验令牌是否存在
- 解析令牌中的用户信息
- 封装用户信息到JSON数据中
- 加密JSON数据
- 将加密后的JSON数据放入到请求头中

好了，经过上述8个步骤，完整的网关已经搭建成功了。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 订单微服务搭建

由于在网关层面已经做了鉴权了（细化到每个URI），因此微服务就不用集成Spring Security单独做权限控制了。

因此这里的微服务也是相对比较简单了，只需要将网关层传递的加密用户信息解密出来，放入到Request中，这样微服务就能随时获取到用户的信息了。

新建一个**oauth2-cloud-order-service**模块，目录如下：

![](https://img.java-family.cn/Spring%20Security/165.png)

新建一个过滤器**AuthenticationFilter**，用于解密网关传递的用户数据，代码如下：

![](https://img.java-family.cn/Spring%20Security/160.png)



新建两个接口，返回当前登录的用户信息，如下：

![](https://img.java-family.cn/Spring%20Security/161.png)

注意：以上两个接口所需要的权限已经放入到了Redis中，权限如下：

- **/order/login/info**：ROLE_admin和ROLE_user都能访问
- **/order/login/admin**：ROLE_admin权限才能访问

![](https://img.java-family.cn/Spring%20Security/162.png)

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 为什么要将URI和权限放入Redis？

在网关的**鉴权管理器**那里是直接从Redis中获取URI对应的权限，然后和令牌中的权限比较，为什么要这样做？

这也是目前企业中比较常用的一种方式，将鉴权完全放在了网关层面，也实现了**动态权限**校验。**当然有些是直接将接口的权限控制在每个微服务中。**

> 采用陈某的这种方案需要另外维护URI和权限的对应关系，当然这种难度很低，便于实现。

只是一种方案，具体是否选用还要考虑到架构层面。

## 测试

同时启动上述三个服务，如下：

![](https://img.java-family.cn/Spring%20Security/166.png)

**1、用密码模式登录user，获取令牌，如下**：

![](https://img.java-family.cn/Spring%20Security/167.png)



**2、使用user用户的令牌访问/order/login/info接口，如下**：

![](https://img.java-family.cn/Spring%20Security/168.png)

可以看到成功返回了，因为具备ROLE_user权限。



**3、使用user用户的令牌访问/order/login/admin接口，如下**：

![](https://img.java-family.cn/Spring%20Security/169.png)

可以看到直接返回了无权限访问，直接在网关层被拦截了。



## 总结

本篇文章只是简单的整合了**网关+OAuth2.0**，实际开发中还有一些细节待完善，由于文章篇幅限制，后续介绍......

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！