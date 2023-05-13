**大家好，我是不才陈某~**

最近订阅[《Spring Cloud Alibaba 项目实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511213&idx=1&sn=9635f9d57390269edef798fa66a03e3b&chksm=fcf77360cb80fa76fe5c9abee83c54fb4e06038d555663c07f5c1be7838bc7abdee5417a8e18&token=1267466554&lang=zh_CN&scene=21#wechat_redirect)视频的朋友又对陈某发问了，如下：

![](https://img.java-family.cn/20220626205627.png)

Spring Security OAuth2的授权码模式一直是个难点，如果你对底层的原理不太理解的话很难去定位到其中的问题

今天这篇文章就针对这位朋友提出的问题做个解答，分为如下三个部分：

1. 授权码模式的登录页面重定制
2. 授权码模式的授权页面重定制
3. 授权码模式的异常页面重定制

关于OAuth2的授权码模式有不理解的可以看陈某之前文章：[妹子始终没搞懂OAuth2.0，今天整合Spring Cloud Security 一次说明白！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502682&idx=1&sn=52a15b623ab6135c134b8262bd605946&chksm=fcf71497cb809d81f1d2dbce76b3e00170f085306b2a2a67a807a6d9e2cf03bf1de3b8f203a2&scene=178&cur_album_id=2151150065472569352#rd)

## 授权码模式的登录页面重定制

下面就以[《Spring Cloud Alibaba 项目实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511213&idx=1&sn=9635f9d57390269edef798fa66a03e3b&chksm=fcf77360cb80fa76fe5c9abee83c54fb4e06038d555663c07f5c1be7838bc7abdee5417a8e18&token=1267466554&lang=zh_CN&scene=21#wechat_redirect)的实战项目来展示一下默认的登录页面什么熊样，如下图：

![](https://img.java-family.cn/20220626211128.png)

是不是有点丑？实际开发中肯定是要根据自己的系统定制这个登录页面

**问题来了：如何定制？**

分为如下几步：

### 1. 定制页面

陈某随便找了一个前端页面`oauth-login.html`，代码如下：

![](https://img.java-family.cn/20220626211534.png)

> 使用thymeleaf进行渲染



### 2. 定义接口跳转

需要在OAuth2的授权服务中定义一个接口跳转到定制的页面，接口如下：

```java
@ApiOperation(value = "表单登录跳转页面")
@GetMapping("/oauth/login")
public String loginPage(Model model){
    //返回跳转页面
    return "oauth-login";
}
```



### 3. Spring Security 中配置

只需要在Spring Security 的表单登录中定义一下跳转的接口即可，代码如下：

![](https://img.java-family.cn/20220626212030.png)

代码解释如下：

1. `loginProcessingUrl`：这个是定义的form表单提交的url
2. `.loginPage`：这个是定义跳转登录页面的url

按照上述三个步骤轻松实现了自定义登录页面，效果如下：

![](https://img.java-family.cn/20220626212836.png)



## 授权码模式的授权页面重定制

下面就以[《Spring Cloud Alibaba 项目实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511213&idx=1&sn=9635f9d57390269edef798fa66a03e3b&chksm=fcf77360cb80fa76fe5c9abee83c54fb4e06038d555663c07f5c1be7838bc7abdee5417a8e18&token=1267466554&lang=zh_CN&scene=21#wechat_redirect)的实战项目来展示一下默认的授权页面什么熊样，如下图：

![](https://img.java-family.cn/20220626213121.png)

那么如何自定义呢？这个自定义就相对麻烦了，需要对Spring Security 底层原理有一定的了解



### 1. 定制页面

陈某随便找了一个页面`oauth-grant.html`，代码如下：

![](https://img.java-family.cn/20220626215529.png)



### 2. 定义接口跳转

授权页面的跳转接口url：`/oauth/confirm_access`，这个接口定义在`org.springframework.security.oauth2.provider.endpoint.WhitelabelApprovalEndpoint`中，如下：

![](https://img.java-family.cn/20220626213404.png)

自定义也很简单，只需要模仿这个接口自定义一个将其覆盖即可，实现如下：

![](https://img.java-family.cn/20220626214929.png)

> **注意**：`@SessionAttributes("authorizationRequest")`这个注解一定要标注，授权请求信息是存储在session中



### 3. 修改默认的映射地址

由于默认的跳转接口是：`/oauth/confirm_access`，陈某刚好定义的接口也是`/oauth/confirm_access`，因此这第3步不用配置也能生效

> 注意：如果你的跳转接口不是`/oauth/confirm_access`，那么需要按照这个步骤配置

修改也很简单，只需要在OAuth2的认证服务的配置类：继承`AuthorizationServerConfigurerAdapter`的配置中修改一下配置，代码如下：

![](https://img.java-family.cn/20220626220659.png)



按照上述3个步骤即可轻松的实现授权页面自定义，效果如下：

![](https://img.java-family.cn/20220626220949.png)



## 授权码模式的异常页面重定制

这个异常页面什么意思呢？授权码的请求url如下：

```sh
http://localhost:9001/blog-auth-server/oauth/authorize?client_id=mugu&response_type=code&scope=all&redirect_uri=http://www.baidu.com
```

假设我将的租户id（`client_id`）修改成数据库中不存在的值，那么将会触犯异常页面，页面如下：

![](https://img.java-family.cn/20220626221505.png)

这个异常页面是不是不太符合系统的要求，肯定是要自定义的



### 1. 定制页面

陈某前端能力有限，没找到现成的，自己随便写了一个`oauth-error.html`，代码如下：

![](https://img.java-family.cn/20220626221910.png)



### 2. 定义接口跳转

这个跳转的接口的逻辑在`AuthorizationEndpoint`中，如下：

![](https://img.java-family.cn/20220626222624.png)

因此只需要重新定义一个接口进行跳转即可，如下：

```java
@ApiOperation(value = "处理授权异常的跳转页面")
@GetMapping("/oauth/error")
public String error(Model model){
    return "oauth-error";
}
```

### 3. 修改默认的映射地址

默认的映射地址为`/oauth/error`，陈某自定义的也是这个，因此第3步可以省略

> 注意：如果你定义的接口不是`/oauth/error`则需要配置

修改也很简单，只需要在OAuth2的认证服务的配置类：继承`AuthorizationServerConfigurerAdapter`的配置中修改一下配置，代码如下：

![](https://img.java-family.cn/20220626223037.png)



按照上述3个步骤即可轻松的实现异常页面自定义，效果如下：

![](https://img.java-family.cn/20220626223333.png)

