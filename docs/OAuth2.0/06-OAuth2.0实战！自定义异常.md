

今天这篇文章主要介绍一下实际工作中使用Spring Security需要定制的一些异常信息。

文章目录如下：

![](https://img.java-family.cn/Spring%20Security/141.png)



## 案例服务搭建

此篇文章沿用上篇文章的认证、资源服务，如下：

**1、认证服务oauth2-auth-server-jwt**

![](https://img.java-family.cn/Spring%20Security/142.png)



**2、资源服务oauth2-auth-resource-jwt**

![](https://img.java-family.cn/Spring%20Security/143.png)



> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 认证服务的异常

先来看一下正确的获取令牌的请求，以**密码模式**为例，如下图：

![](https://img.java-family.cn/Spring%20Security/106.png)

密码模式需要传递**5**个参数，分别是**用户名**、**密码**、**客户端id**，**客户端秘钥**、**授权类型**。

**那么问题来了：如果任意一个参数传错了，返回什么？**

### **1、用户名、密码错误**

故意输错用户名或者密码，返回信息如下：

![](https://img.java-family.cn/Spring%20Security/107.png)



### **2、授权类型错误**

输入一个不存在的授权类型，返回信息如下：

![](https://img.java-family.cn/Spring%20Security/108.png)



### **3、客户端ID，秘钥错误**

输入错误的客户端id或者秘钥，返回信息如下：

![](https://img.java-family.cn/Spring%20Security/109.png)



**感觉如何？是不是很酸爽？很显然这返回的信息不适合前后端交互，别着急，下面介绍解决方案**

## 认证服务自定义异常信息

上面列举了三种常见的异常，解决方案实际可以分为两种：

- 用户名，密码错误异常、授权类型异常
- 客户端ID、秘钥异常

**陈某这里针对这两种异常先上解决方案，后面再从源码解释为什么这么做？**

### **1、用户名，密码错误异常、授权类型异常**

针对用户名、密码、授权类型错误的异常解决方式比较复杂，需要定制的比较多。

**1、定制提示信息、响应码**

这部分根据自己业务需要定制，陈某这里只是给出个例子，代码如下：

![](https://img.java-family.cn/Spring%20Security/110.png)



**2、自定义WebResponseExceptionTranslator**

需要自定义一个异常翻译器，默认的是**DefaultWebResponseExceptionTranslator**，此处必须重写，其中有一个需要实现的方法，如下：

```java
ResponseEntity<T> translate(Exception e) throws Exception;
```

这个方法就是根据传递过来的**Exception**判断不同的异常返回特定的信息，这里需要判断的异常的如下：

- **UnsupportedGrantTypeException**：不支持的授权类型异常
- **InvalidGrantException**：用户名或者密码错误的异常

创建一个**OAuthServerWebResponseExceptionTranslator**实现**WebResponseExceptionTranslator**，代码如下：

![](https://img.java-family.cn/Spring%20Security/111.png)



**3、认证服务配置文件中配置**

需要将自定义的异常翻译器**OAuthServerWebResponseExceptionTranslator**在配置文件中配置，很简单，一行代码的事。

在**AuthorizationServerConfig**配置文件指定，代码如下：

![](https://img.java-family.cn/Spring%20Security/112.png)

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

**4、测试**

按照上述的配置完成后，测试下用户名、密码错误、授权类型错误是否能够正确返回定制的提示信息，如下：

![用户名、密码错误](https://img.java-family.cn/Spring%20Security/113.png)

![授权类型不支持](https://img.java-family.cn/Spring%20Security/114.png)



**5、源码追踪**

实践有了，总该理解一下为什么这么做吧？下面从源码的角度告诉你为什么要这么做？

我们知道获取令牌的接口为 **/oauth/token**，这个接口定义在**TokenEndpoint#postAccessToken()**（POST请求）方法中，如下图：

![](https://img.java-family.cn/Spring%20Security/115.png)

**我们先不看其中的逻辑，平时我们写接口的异常怎么处理？**

一般都是通过 **@ExceptionHandler** 特定的异常，然后统一的进行异常处理，是不是这样？

然后看一下上述的两种异常属于什么类型的，如下：

![UnsupportedGrantTypeException](https://img.java-family.cn/Spring%20Security/116.png)

![InvalidGrantException](https://img.java-family.cn/Spring%20Security/117.png)

是不是都继承了**OAuth2Exception**，那么尝试在**TokenEndpoint**这个类中找找有没有处理**OAuth2Exception**这个异常的处理器，果然找到了一个 **handleException()** 方法，如下：

![](https://img.java-family.cn/Spring%20Security/118.png)

可以看到，这里的异常翻译器已经使用了我们自定义的**OAuthServerWebResponseExceptionTranslator**。可以看下默认的异常翻译器是啥，代码如下：

![](https://img.java-family.cn/Spring%20Security/119.png)

看到没，就是这个**DefaultWebResponseExceptionTranslator**

问题又来了：**为什么在配置文件中设置了OAuthServerWebResponseExceptionTranslator就会生效呢？**

这个不得不看下 **@EnableAuthorizationServer** 这个注解了，源码如下：

![](https://img.java-family.cn/Spring%20Security/120.png)

注入了这个**AuthorizationServerEndpointsConfiguration**配置类，其中注入了**AuthorizationEndpoint**这个bean，如下：

![](https://img.java-family.cn/Spring%20Security/121.png)

将自定义的异常翻译器设置进入了**AbstractEndpoint**这个抽象类中，而**TokenEndpoint**正是继承了这个抽象类，复用了其中的异常翻译器，代码如下：

![](https://img.java-family.cn/Spring%20Security/122.png)

哦了，问题解决了，学东西一定要知其所以然...............



### **2、客户端ID、秘钥异常**

这部分比较复杂，想要理解还是需要些基础的，解决这个异常的方案很多，陈某只是介绍其中一种，下面详细介绍。

**1、定制提示信息、响应码**

这部分根据自己业务需要定制，陈某这里只是给出个例子，代码如下：

![](https://img.java-family.cn/Spring%20Security/110.png)



**2、自定义AuthenticationEntryPoint**

这个**AuthenticationEntryPoint**是不是很熟悉，前面的文章已经介绍过了，此处需要自定义来返回定制的提示信息。

创建**OAuthServerAuthenticationEntryPoint**，实现AuthenticationEntryPoint，重写其中的方法，代码如下：

![](https://img.java-family.cn/Spring%20Security/126.png)



**3、改造ClientCredentialsTokenEndpointFilter**

**ClientCredentialsTokenEndpointFilter**这个过滤器的主要作用就是校验客户端的ID、秘钥，代码如下：

![](https://img.java-family.cn/Spring%20Security/125.png)

有几个重要的部分需要讲一下，如下：

- 构造方法中需要传入第2步自定义的 **OAuthServerAuthenticationEntryPoint**
- 重写 **getAuthenticationManager()** 方法返回IOC中的AuthenticationManager
- 重写**afterPropertiesSet()** 方法，用于自定义认证失败、成功处理器，失败处理器中调用**OAuthServerAuthenticationEntryPoint**进行异常提示信息返回



**4、OAuth配置文件中指定过滤器**

只需要将自定义的过滤器添加到**AuthorizationServerSecurityConfigurer**中，代码如下：

![](https://img.java-family.cn/Spring%20Security/127.png)

第**①**部分是添加过滤器，其中**authenticationEntryPoint**使用的是第2步自定义的**OAuthServerAuthenticationEntryPoint**

第**②**部分一定要注意：一定要去掉这行代码，具体原因源码解释。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

**5、测试**

直接输入错误的秘钥，结果如下：

![](https://img.java-family.cn/Spring%20Security/128.png)



**6、源码追踪**

**1、OAuthServerAuthenticationEntryPoint在何时调用？**

OAuthServerAuthenticationEntryPoint这个过滤器继承了 **AbstractAuthenticationProcessingFilter** 这个抽象类，一切的逻辑都在 **doFilter()** 中，陈某简化了其中的关键代码如下：

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
    try {
        	//调用子类的attemptAuthentication方法，获取参数并且认证
			authResult = attemptAuthentication(request, response);
		}
		catch (InternalAuthenticationServiceException failed) {
            //一旦认证异常，则调用unsuccessfulAuthentication方法，通过failureHandler处理
			unsuccessfulAuthentication(request, response, failed);
			return;
		}
		catch (AuthenticationException failed) {
            //一旦认证异常，则调用unsuccessfulAuthentication方法，通过failureHandler处理
			unsuccessfulAuthentication(request, response, failed);
			return;
		}
		//认证成功，则调用successHandler处理
		successfulAuthentication(request, response, chain, authResult);
}
```

关键代码在 **unsuccessfulAuthentication()** 这个方法中，代码如下：

![](https://img.java-family.cn/Spring%20Security/129.png)



**2、自定义的过滤器如何生效的？**

这个就要看 **AuthorizationServerSecurityConfigurer#configure()** 这个方法了，其中有一段代码如下：

![](https://img.java-family.cn/Spring%20Security/130.png)

这段代码就是遍历添加的过滤器将其添加到**过滤器链**中，在**BasicAuthenticationFilter**这个过滤器之前。

添加到security的过滤器链中，这个过滤器自然会生效了。

**3、为什么不能加.allowFormAuthenticationForClients()？**

还是在 **AuthorizationServerSecurityConfigurer#configure()** 这个方法中，一旦设置了 **allowFormAuthenticationForClients** 为true，则会创建 **ClientCredentialsTokenEndpointFilter**，此时自定义的自然失效了。

![](https://img.java-family.cn/Spring%20Security/132.png)



## 资源服务器的异常

从认证服务获取到令牌之后去请求资源服务的资源，这里涉及到的异常主要有两个，如下：

### **1、令牌失效**

比如令牌不正确、过期，此时返回的异常提示如下：

![](https://img.java-family.cn/Spring%20Security/133.png)

### **2、权限不足**

令牌的权限不足，比如 **/admin** 接口只允许 **admin** 角色访问，此时返回的异常信息如下：

![](https://img.java-family.cn/Spring%20Security/134.png)



## 资源服务自定义异常信息

下面针对上述两种异常分别定制异常提示信息，这个比认证服务定制简单。

### 1、令牌失效

这个比较简单，也是需要自定义**AuthenticationEntryPoint**。步骤如下：

**1、自定义AuthenticationEntryPoint**

这个和认证服务的客户端异常类似，这里不再详细说了，直接贴代码，如下：

![](https://img.java-family.cn/Spring%20Security/135.png)



**2、OAuth配置文件中配置**

这个比较简单，直接在配置文件中配置即可，代码如下：

![](https://img.java-family.cn/Spring%20Security/136.png)



**3、测试**

此时拿着失效的令牌访问资源服务，可以看到已经正常返回定制的提示信息了，如下：

![](https://img.java-family.cn/Spring%20Security/137.png)



源码和认证服务的类似，自己断点试试，还是很简单的。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

### 2、权限不足

这个异常定制就更简单了，陈某在第一篇文章：[实战！Spring Boot Security+JWT前后端分离架构登录认证！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502546&idx=1&sn=bfb6fd9d96d8c5bf107a4981ba5e1547&chksm=fcf7151fcb809c09b7ae29de8c0af0d00976539a46ee5f9bf583a6a7b196ea82f26ce98fd982&token=1622810645&lang=zh_CN#rd)介绍过，下面简单的贴下代码。

**1、自定义AccessDeniedHandler**

代码如下：

![](https://img.java-family.cn/Spring%20Security/138.png)



**2、OAuth配置文件中配置**

和令牌失效的异常配置在同一个方法中，代码如下：

![](https://img.java-family.cn/Spring%20Security/139.png)



**3、测试**

访问 **/admin** 接口，此时的提示信息如下：

![](https://img.java-family.cn/Spring%20Security/140.png)



> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！













