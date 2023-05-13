

今天就来介绍一下如何在令牌中添加业务所需的额外信息，称之为令牌的**增强**。

文章目录如下：

![](https://img.java-family.cn/Spring%20Security/182.png)

## 网关层如何获取令牌中的信息？

在上篇文章**网关集成OAuth2.0**中定义了一个全局过滤器**GlobalAuthenticationFilter**用来解析令牌并且封装用户信息传递给下游微服务，关键代码如下：

![](https://img.java-family.cn/Spring%20Security/172.png)

其中令牌的关键信息都存放在**additionalInformation**中，默认的有三个，分别是用户名、权限、jti，如下：

![](https://img.java-family.cn/Spring%20Security/173.png)

很显然这三个信息实际开发中并不够用，因此必须要对**additionalInformation**进行扩展。



## 令牌如何增强？

Spring Security中的存储用户信息的就是 **UserDetails** 这个实体类，认证过程中通过 **UserDetailsService#loadUserByUsername()** 方法从数据库中加载用户信息。

前面我们定义过**SecurityUser**实现了UserDetails这个接口，如下：

![](https://img.java-family.cn/Spring%20Security/174.png)

默认只有这个三个字段，显然这些是不够用的，必须要对其扩展。

**需求**：现在下游微服务需要**userId**查询订单。

因此必须在认证过程中就需要将用户的userId存放到令牌中，这样在网关层才能从令牌中获取这部分信息，然后解析传递给下游微服务。

> 注意：JWT令牌中的信息能够被解析出来，因此不要存放用户的敏感信息

### **1、扩展SecurityUser**

需要在这个实体类中添加userId这个字段，如下：

![](https://img.java-family.cn/Spring%20Security/175.png)



### **2、UserDetailsService中查询出userId**

在**loadUserByUsername()**这个方法中要从数据库中查询出**userId**封装到**SecurityUser**中，如下：

![](https://img.java-family.cn/Spring%20Security/176.png)



### **3、改造JwtAccessTokenConverter**

新建一个类**JwtAccessTokenEnhancer**，继承**JwtAccessTokenConverter**，重写其中的 **enhance()** 方法，代码如下：

![](https://img.java-family.cn/Spring%20Security/177.png)

逻辑很简单，分为四步：

- 从**OAuth2Authentication**中获取认证成功的用户信息**SecurityUser**
- 新建一个**LinkedHashMap**存放额外的信息。
- 将额外信息添加到**additionalInformation**中
- 调用父类的 **enhance()** 方法

### **4、注入JwtAccessTokenConverter**

此时在**AccessTokenConfig**中注入的**JwtAccessTokenConverter**就直接使用**JwtAccessTokenEnhancer**即可，代码如下：

![](https://img.java-family.cn/Spring%20Security/178.png)



经过以上4个步骤，认证中心的代码已经改好了，最重要就是重写**JwtAccessTokenConverter**中的 **enhance()** 方法，在其中添加额外信息。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 过滤器改造

这里需要改造两个过滤器，分别如下：

- **网关层的全局过滤器**：将添加的额外信息解析放入到请求头中
- **微服务的过滤器**：从请求头取出信息。

### **1、网关层的全局过滤器**

在网关的全局过滤器**GlobalAuthenticationFilter**中的添加两行代码，如下：

![](https://img.java-family.cn/Spring%20Security/179.png)

很简单，就是从令牌的additionalInformation中取出然后放入到JSON数据中。



### **2、微服务过滤器改造**

在微服务的过滤器**AuthenticationFilter**中添加两行代码，如下：

![](https://img.java-family.cn/Spring%20Security/180.png)

从解密的**JOSN**数据取出**userId**封装到**LoginVal**中，方便业务方法直接获取。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 总结

令牌增强的思路很简单，大致如下：

1. 重写**认证服务**的**JwtAccessTokenConverter#enhance()**方法，将额外信息放入到**additionalInformation**中。
2. **网关层**通过**全局过滤器**从令牌的**additionalInformation**解析出额外信息，封装到JSON数据中，放入请求头中
3. **微服务层**解析出请求头中的JSON数据，取出额外信息重新封装，方便业务方法直接获取。

案例源码中涉及到的文件修改如下：

![](https://img.java-family.cn/Spring%20Security/181.png)

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！
