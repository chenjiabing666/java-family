**大家好，我是不才陈某~**


本篇文章介绍一下Spring Security如何扩展新的授权类型，也是实际开发中非常重要的知识点。

目录如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/13.png)



## 为什么需要自定义授权类型？

前面介绍OAuth2.0的基础知识点时介绍过支持的4种授权类型，分别如下：

- 授权码模式
- 简化模式
- 客户端模式
- 密码模式

关于上述4种授权类型不清楚的，可以看之前的文章：[妹子始终没搞懂OAuth2.0，今天整合Spring Cloud Security 一次说明白！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502682&idx=1&sn=52a15b623ab6135c134b8262bd605946&chksm=fcf71497cb809d81f1d2dbce76b3e00170f085306b2a2a67a807a6d9e2cf03bf1de3b8f203a2&scene=178&cur_album_id=2042874937312346114#rd)

实际生产中上述四种授权类型根本不够用，比如常见的授权类型如下：

- 微信认证
- QQ认证
- 手机号+验证码认证
- 图形验证码认证
- 邮箱认证

因此我们必须懂得OAuth2.0如何自定义授权类型，这也是本篇文章的重点。



## 实现思路

Spring Security 定制授权类型其实很简单，主要是掌握其中的思路，下面是密码模式的授权流程，如下图：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/1.jpg)

根据上述流程图可以跟着源码进去看看，不难发现有几个如下重要点：

- 每种授权类型都对应一个实现类**TokenGranter**，其中定义着授权类型
- 所有 `TokenGranter` 实现类都通过 `CompositeTokenGranter` 中的 `tokenGranters` 集合存起来。
- 然后通过判断 `grantType` 参数来定位具体使用那个 `TokenGranter` 实现类来处理授权。
- 每种授权方式都对应一个**AuthenticationProvider**
- `TokenGranter` 类会 new 一个 **AuthenticationToken**实现类，如 `UsernamePasswordAuthenticationToken` 传给 `ProviderManager` 类。



因此想要自定义一个授权类型，必须构建自己的**TokenGranter**、**AuthenticationProvider**、**AuthenticationToken**。



## 代码实现

下面就以**手机号+密码**的登录方式定义一个类型：**mobile_pwd**，剩下的自己照葫芦画瓢。



### 1、自定义UserDetailService

这个和密码授权类型类似，要实现一个方法从数据库中根据手机号查询用户的详细信息。

定义一个**SmsCodeUserDetailService**接口如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/2.png)



主要就是一个 **loadUserByMobile()** 方法，实现类如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/3.png)



### 2、自定义AuthenticationToken

类似于密码模式的中**UsernamePasswordAuthenticationToken**，自定义一个**MobilePasswordAuthenticationToken**封装手机号和密码，如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/4.png)



### 3、自定义TokenGranter

每种授权类型都对应一种TokenGranter，其中会定义授权类型的名称，比如密码模式的**ResourceOwnerPasswordTokenGranter**，其中的GRANT_TYPE为password。

自定义一个**MobilePwdGranter**，照葫芦画瓢，模仿着改改，代码如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/5.png)



### 4、自定义AuthenticationProvider

这个类就是真正的处理类，经过TokenGranter后，会找到对应的AuthenticationProvider，然后取出参数从数据库（**UserDetailService**）中查询对应的信息进行匹配。

自定义**MobilePasswordAuthenticationProvider**，代码如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/6.png)![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/7.png)

> 案例源码已上传GitHub，关注公众号：**码猿技术专栏**，回复关键：**9529** 获取！

### 5、将自定义的MobilePasswordAuthenticationProvider注入IOC容器

这里必须将自定义的MobilePasswordAuthenticationProvider注入到IOC容器，如果不注入，会报找不到能处理的AuthenticationProvider这个异常。

新建**SmsCodeSecurityConfig**，代码如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/8.png)



> 注意：由于使用的外部配置，因此必须在全局配置中指定

### 6、Security的全局配置指定SmsCodeSecurityConfig

由于是分开配置，因此必须在全局配置中指定才会生效，代码如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/9.png)



### 7、加到CompositeTokenGranter集合中

需要将自定义的授权类型加到集合CompositeTokenGranter中，此处需要修改认证中心的配置类（**AuthorizationServerConfig**）中的代码，如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/10.png)



### 8、oauth_client_details表中添加授权类型

**oauth_client_details**这个表是存储客户端的详细信息的，需要在对应的客户端资源那一行中的**authorized_grant_types**这个字段中添加自定义的授权类型，多个用逗号分隔。

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/11.png)



## 测试

经过上述的步骤已经配置完成，下面来测试，启动服务，请求如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/12.jpg)



## 源码获取

授权类型主要是针对 **认证中心（oauth2-cloud-auth-server）** 的改动，改动的目录如下：

![](https://img.java-family.cn/%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%88%E6%9D%83%E7%B1%BB%E5%9E%8B/12.png)

陈某直接在之前网关整合**Spring Security**的源码上更改了一版。

> 案例源码已上传GitHub，关注公众号：**码猿技术专栏**，回复关键：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！



