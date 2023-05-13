## 什么是JWT？

OAuth2.0体系中令牌分为两类，分别是**透明令牌**、**不透明令牌**。

不透明令牌则是令牌本身不存储任何信息，比如一串**UUID**，上篇文章中使用的**InMemoryTokenStore**就类似这种。

因此资源服务拿到这个令牌必须调调用**认证授权服务**的接口进行令牌的**校验**，高并发的情况下**延迟很高，性能很低**，正如上篇文章中资源服务器中配置的校验，如下：

![](https://img.java-family.cn/Spring%20Security/89.png)

透明令牌本身就存储这部分用户信息，比如**JWT**，资源服务可以**调用自身的服务对该令牌进行校验解析**，不必调用认证服务的接口去校验令牌。

JWT相信大家都有了解，分为三部分，分别是**头部**、**载荷**、**签名**，如下：

```json
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJhdWQiOlsicmVzMSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTYzODYwNTcxOCwiYXV0aG9yaXRpZXMiOlsiUk9MRV91c2VyIl0sImp0aSI6ImRkNTVkMjEzLThkMDYtNGY4MC1iMGRmLTdkN2E0YWE2MmZlOSIsImNsaWVudF9pZCI6Im15anN6bCJ9.
koup5-wzGfcSVnaaNfILwAgw2VaTLvRgq2JVnIHYe_Q
```

**头部**定义了JWT基本信息，如类型和签名算法。

**载荷**包含了一些基本信息（签发时间、过期时间.....），另外还可以添加一些自定义的信息，比如用户的部分信息。

**签名**部分将前两个字符串用 . 连接后，使用头部定义的加密算法，利用密钥进行签名，并将签名信息附在最后。

## OAuth2.0认证授权服务搭建

OAuth2.0分为认证授权中心、资源服务，认证中心用于颁发令牌，资源服务解析令牌并且提供资源。

### 1、案例架构

新建**oauth2-auth-server-jwt**模块，沿用上篇文章[妹子始终没搞懂OAuth2.0，今天整合Spring Cloud Security 一次说明白！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502682&idx=1&sn=52a15b623ab6135c134b8262bd605946&chksm=fcf71497cb809d81f1d2dbce76b3e00170f085306b2a2a67a807a6d9e2cf03bf1de3b8f203a2&token=404376711&lang=zh_CN#rd)的代码，在其之上做些修改，目录如下：

![](https://img.java-family.cn/Spring%20Security/90.png)

### 2、令牌配置

令牌相关的配置都放在了**AccessTokenConfig**这个配置类中，代码如下：

![](https://img.java-family.cn/Spring%20Security/91.png)



**1、 JwtAccessTokenConverter**

令牌增强类，用于JWT令牌和OAuth身份进行转换

**2、TokenStore**

令牌的存储策略，这里使用的是**JwtTokenStore**，使用JWT的令牌生成方式，其实还有以下两个比较常用的方式：

- **RedisTokenStore**：将令牌存储到Redis中，此种方式相对于内存方式来说性能更好
- **JdbcTokenStore**：将令牌存储到数据库中，需要新建从对应的表，有兴趣的可以尝试

**3、SIGN_KEY**

JWT签名的秘钥，这里使用的是对称加密，**资源服务中也要使用相同的秘钥进行校验和解析JWT令牌**。

> **注意**：实际工作中还是要使用非对称加密的方式，比较安全，这种方式后续文章介绍。



### 3、令牌管理服务的配置

这个放在了**AuthorizationServerConfig**这个配置类中，代码如下：

![](https://img.java-family.cn/Spring%20Security/92.png)

使用的是DefaultTokenServices这个实现类，其中可以配置令牌相关的内容，比如**access_token**、**refresh_token**的过期时间，默认时间分别为12小时、30天。

最重要的一行代码当然是设置令牌增强，使用JWT方式生产令牌，如下：

```java
services.setTokenEnhancer(jwtAccessTokenConverter);
```

### 4、令牌访问端点添加tokenServices

在**AuthorizationServerEndpointsConfigurer**中添加这个令牌服务，代码如下：

![](https://img.java-family.cn/Spring%20Security/93.png)

好了，至此认证中心的JWT令牌生成方式配置完成了.........

> 案例源码已经上传GitHub，关注公众号：码猿技术专栏，回复关键词 **9529** 获取！



## OAuth2.0资源服务搭建

资源服务搭建非常简单了，配置一个JWT令牌校验服务即可。

### 1、案例架构

新建一个**oauth2-auth-resource-jwt**模块，目录如下：

![](https://img.java-family.cn/Spring%20Security/94.png)

### 2、令牌配置

直接复用授权服务的**AccessTokenConfig**，由于资源服务需要校验解析JWT令牌，因此直接复用即可，代码如下：

![](https://img.java-family.cn/Spring%20Security/91.png)

> **注意**：这里的JWT加密的秘钥一定要和认证中心的一样。

### 3、配置令牌服务

生成的**ResourceServerTokenServices**对象，其中使用JWT令牌增强，如下：

![](https://img.java-family.cn/Spring%20Security/95.png)



### 4、资源ID和令牌校验服务配置

将**资源id**和**令牌服务**配置到**ResourceServerSecurityConfigurer**中，代码如下：

![](https://img.java-family.cn/Spring%20Security/96.png)

由于使用了JWT这种透明令牌，令牌本身携带着部分用户信息，因此不需要通过远程调用认证中心的接口校验令牌。

> 案例源码已经上传GitHub，关注公众号：码猿技术专栏，回复关键词 **9529** 获取！

## 测试

下面通过获取令牌、调用资源进行测试逻辑是否走通。

**1、使用密码模式获取令牌**

POSTMAN请求如下：

![](https://img.java-family.cn/Spring%20Security/97.png)

可以看到已经成功返回了JWT令牌。



**2、携带令牌调用资源服务**

直接拿着获取的**access_token**调用资源服务的接口，请求如下：

![](https://img.java-family.cn/Spring%20Security/98.png)

好了，JWT令牌测试成功............



## 源码追踪

源码中最重要的部分当然是获取令牌、校验令牌这两个流程了，小陈某下面详细说说。

### 1、获取令牌

获取令牌就比较简简单了，当然从接口 **/oauth/token**入手了，这个接口在`TokenEndpoint#postAccessToken()`方法中，如下图：

![](https://img.java-family.cn/Spring%20Security/99.png)

这个方法中有两个关键步骤，如下：

**1、根据clientId加载客户端信息**

这一步是从根据客户端传入的clientId获取客户端的详细信息，代码如下：

```java
ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);
```

这里的**ClientDetailsService**有两类，如下：

- **InMemoryClientDetailsService**：客户端配置存储在内存中，本篇文章所使用的便是这个
- **JdbcClientDetailsService**：客户端配置存储在数据库中，**后续文章介绍**。

**2、生成OAuth2AccessToken返回客户端**

**OAuth2AccessToken**是封装的返回对象，**/oauth/token**这个接口的作用就是将令牌封装到OAuth2AccessToken返回，代码如下：

```java
OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
```

**getTokenGranter()**：获取授权类型，比如密码类型、授权码类型

**grant()**：这个方法则是真正的业务方法，其中调用**DefaultTokenServices#createAccessToken()** 方法生成令牌。

**DefaultTokenServices**这个还记得吗，令牌服务，在**AuthorizationServerConfig**配置文件配置的，如下：

![](https://img.java-family.cn/Spring%20Security/100.png)



**createAccessToken()**方法内部真正的业务方法其实是**JwtAccessTokenConverter#enhance()**，内部生成JWT令牌，封装进入**OAuth2AccessToken**对象返回，方法如下：

![](https://img.java-family.cn/Spring%20Security/101.png)

**JwtAccessTokenConverter**这个还记得吗？令牌增强类，在**AccessTokenConfig**这个配置文件中配置的，如下：

![](https://img.java-family.cn/Spring%20Security/102.png)

主流程图如下：

![](https://img.java-family.cn/Spring%20Security/103.png)

### 2、校验令牌

校验令牌的更加简单了，入口就在**OAuth2AuthenticationProcessingFilter**这个过滤器，内部会调用**OAuth2AuthenticationManager**中的**authenticate()**方法进行验证令牌。

校验主流程如下：

![](https://img.java-family.cn/Spring%20Security/104.png)

自己debug跟着源码试试吧............

> 案例源码已经上传GitHub，关注公众号：码猿技术专栏，回复关键词 **9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！







