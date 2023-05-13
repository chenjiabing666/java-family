
今天这篇文章介绍一下如何在**修改密码**、**修改权限**、**注销**等场景下使JWT失效。

文章的目录如下：

![](https://img.java-family.cn/Spring%20Security/195.png)

## 解决方案

JWT最大的一个优势在于它是**无状态**的，自身包含了认证鉴权所需要的所有信息，服务器端无需对其存储，从而给服务器减少了存储开销。

但是无状态引出的问题也是可想而知的，它无法作废未过期的JWT。举例说明注销场景下，就传统的**cookie/session**认证机制，只需要把存在服务器端的session删掉就OK了。

但是JWT呢，它是不存在服务器端的啊，好的那我删存在客户端的JWT行了吧。额，社会本就复杂别再欺骗自己了好么，被你在客户端删掉的JWT还是可以通过服务器端认证的。

使用JWT要非常明确的一点：**JWT失效的唯一途径就是等待时间过期**。

但是可以借助外力保存JWT的状态，这时就有人问了：你这不是打脸吗？用JWT就因为它的无状态性，这时候又要保存它的状态？

其实不然，这不被逼上梁山了吗？不使用外力保存JWT的状态，你说如何实现注销失效？

常用的方案有两种，**白名单**和**黑名单**方式。

### **1、白名单**

白名单的逻辑很简单：认证通过时，将JWT存入redis中，注销时，将JWT从redis中移出。这种方式和cookie/session的方式大同小异。

### **2、黑名单**

黑名单的逻辑也非常简单：注销时，将JWT放入redis中，并且设置过期时间为JWT的过期时间；请求资源时判断该JWT是否在redis中，如果存在则拒绝访问。

白名单和黑名单这两种方案都比较好实现，但是**黑名单带给服务器的压力远远小于白名单**，毕竟注销不是经常性操作。

## 黑名单方式实现

下面以黑名单的方式介绍一下如何在网关层面实现JWT的注销失效。

**究竟向Redis中存储什么？**

如果直接存储JWT令牌可行吗？当然可行，不过JWT令牌可是很长的哦，这样对内存的要求也是挺高的。

熟悉JWT令牌的都知道，JWT令牌中有一个**jti**字段，这个字段可以说是JWT令牌的唯一ID了，如下：

![](https://img.java-family.cn/Spring%20Security/183.png)

因此可以将这个**jti**字段存入redis中，作为唯一令牌标识，这样一来是不是节省了很多的内存？

**如何实现呢？** 分为两步：

1. 网关层的全局过滤器中需要判断黑名单是否存在当前JWT
2. 注销接口中将JWT的**jti**字段作为**key**存放到redis中，且设置了JWT的过期时间

### **1、网关层解析JWT的jti、过期时间放入请求头中**

在网关的全局过滤器**GlobalAuthenticationFilter**中直接从令牌中解析出**jti**和**过期时间**。

这里的逻辑分为如下步骤：

1. 解析JWT令牌的jti和过期时间
2. 根据jti从redis中查询是否存在黑名单中，如果存在则直接拦截，否则放行
3. 将解析的jti和过期时间封装到JSON中，传递给下游微服务

关键代码如下：

![](https://img.java-family.cn/Spring%20Security/184.png)



### **2、下游微服务的过滤器修改**

还记得上篇文章：[实战干货！Spring Cloud Gateway 整合 OAuth2.0 实现分布式统一认证授权！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247503249&idx=1&sn=b33ae3ff70a08b17ee0779d6ccb30b53&chksm=fcf7125ccb809b4aa4985da09e620e06c606754e6a72681c93dcc88bdc9aa7ba0cb64f52dbc3&token=1286998820&lang=zh_CN#rd)中微服务的过滤器**AuthenticationFilter**吗？

**AuthenticationFilter**这个过滤器用来解密网关层传递的JSON数据，并将其封装到Request中，这样在业务方法中便可以随时获取到想要的用户信息。

这里我是把JWT相关的信息同时封装到了**Request**中，实体类为**JwtInformation**，如下：

![](https://img.java-family.cn/Spring%20Security/185.png)

**LoginVal**继承了**JwtInformation**，如下：

![](https://img.java-family.cn/Spring%20Security/186.png)

此时**AuthenticationFilter**这个过滤器修改起来就很简单了，只需要将**jti**和过期时间封装到**LoginVal**中即可，关键代码如下：

![](https://img.java-family.cn/Spring%20Security/187.png)

逻辑很简单，上图都有标注。

### **3、注销接口实现**

之前文章中并没有提供注销接口，因为无状态的JWT根本不需要退出登录，傻等着过期呗。

当然为了实现注销登录，借助了Redis，那么注销接口必不可少了。

逻辑很简单，直接将退出登录的JWT令牌的jti设置到Redis中，过期时间设置为JWT过期时间即可。代码如下：

![](https://img.java-family.cn/Spring%20Security/188.png)

OK了，至此已经实现了JWT注销登录的功能.......

> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

涉及到的三个模块的改动，分别如下：

| 名称                     | 功能               |
| ------------------------ | ------------------ |
| oauth2-cloud-auth-server | OAuth2.0认证授权服 |
| oauth2-cloud-gateway     | 网关服务           |
| oauth2-cloud-auth-common | 公共模块           |

![](https://img.java-family.cn/Spring%20Security/194.png)

## 总结

思想很简单，JWT既然是无状态的，只能借助Redis记录它的状态，这样才能达到使其失效的目的。

## 测试

业务基本完成了，下面走一个流程测试一下，如下：

**1、登录，申请令牌**

![](https://img.java-family.cn/Spring%20Security/189.png)



**2、拿着令牌访问接口**

该令牌并没有注销，因此可以正常访问，如下：

![](https://img.java-family.cn/Spring%20Security/190.png)



**3、调用接口注销登录**

请求如下：

![](https://img.java-family.cn/Spring%20Security/191.png)



**4、拿着注销的令牌访问接口**

由于令牌已经注销了，因此肯定访问不通接口，返回如下：

![](https://img.java-family.cn/Spring%20Security/192.png)

> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！









