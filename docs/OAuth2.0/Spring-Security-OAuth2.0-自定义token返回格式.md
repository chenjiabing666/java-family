**大家好，我是不才陈某~**

最近订阅[《Spring Cloud Alibaba 项目实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511213&idx=1&sn=9635f9d57390269edef798fa66a03e3b&chksm=fcf77360cb80fa76fe5c9abee83c54fb4e06038d555663c07f5c1be7838bc7abdee5417a8e18&token=1267466554&lang=zh_CN&scene=21#wechat_redirect)的朋友针对Spring Security OAuth2.0 想要陈某补充一些知识，如下：

![](https://img.java-family.cn/20220618164927.png)

今天这篇文章就来回答其中一个问题：**如何自定义token的返回格式？**



## 问题描述

Spring Security OAuth的token返回格式都是默认的，但是往往这个格式是不适配系统，`/oauth/token`返回的格式如下：

```json
{
    "access_token": token
    "token_type": "bearer",
    "refresh_token": xxxx
    "expires_in": xxx,
    "scope": "xxx",
    "jti": xxxx
    ....................
}
```

然而此时系统中的统一返回格式为：

```json
{
    "code":xxx
    "data":xxx
    "msg":xxx
}
```

那么如何去对默认的格式进行修改呢？

## 解决方案

其实解决方案还是很多的，据陈某了解有如下两种解决方案：

1. 使用AOP的方式对`/oauth/token`这个接口的结果拦截修改
2. 重定义接口覆盖默认的

第一种方案呢可以实现，但是对于陈某来说不够优雅，实现比较简单，不显逼格

于是陈某今天介绍第二种方案，一种比较优雅的方式；想要理解第二种方式必须对Spring Security的底层源码有一些了解。

`/oauth/token`这个接口定义在哪里呢？通过源码我们知道定义在`org.springframework.security.oauth2.provider.endpoint.TokenEndpoint`中，如下：

```java
@RequestMapping(value = "/oauth/token", method=RequestMethod.GET)
public ResponseEntity<OAuth2AccessToken> getAccessToken(Principal principal, @RequestParam
Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {}

@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {}
```

可以看到针对这个接口定义了两个，一个是GET请求、一个是POST请求

`TokenEndpoint`其实就是一个接口，使用注解`@FrameworkEndpoint`标注，这个注解和`@Controller`的作用一样，如下：

```java
@FrameworkEndpoint
public class TokenEndpoint extends AbstractEndpoint {}
```

那么知道在哪里定义的就好办了，模仿着它这个接口自己重新定义一个覆盖掉不就好了，如下：

```java
@Api(value = "OAuth接口")
@RestController
@RequestMapping("/oauth")
@Slf4j
public class AuthController implements InitializingBean {

    //令牌请求的端点
    @Autowired
    private TokenEndpoint tokenEndpoint;

    //自定义异常翻译器，针对用户名、密码异常，授权类型不支持的异常进行处理
    private OAuthServerWebResponseExceptionTranslator translate;

    /**
     * 重写/oauth/token这个默认接口，返回的数据格式统一
     */
    @PostMapping(value = "/token")
    public ResultMsg<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
            Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
        OAuth2AccessToken accessToken = tokenEndpoint.postAccessToken(principal, parameters).getBody();
        return ResultMsg.resultSuccess(accessToken);
    }
}
```

可以看到接口内部不需要自己重写逻辑，只需要调用`TokenEndpoint`中的方法

> 注意：由于对TokenEndpoint中的端点重写了，因此前面定义的对用户名、密码之类的异常捕获的翻译类（**OAuthServerWebResponseExceptionTranslator**）将会失效，需要在全局异常中进行捕获

上面是`/oauth/token`的接口，`/oauth/check_token`这个校验token的接口如需自定义也是可以的，对应的类是`org.springframework.security.oauth2.provider.endpoint.CheckTokenEndpoint`

重写后代码如下：

```java
@Api(value = "OAuth接口")
@RestController
@RequestMapping("/oauth")
@Slf4j
public class AuthController implements InitializingBean {

    @Autowired
    private CheckTokenEndpoint checkTokenEndpoint;

    //自定义异常翻译器，针对用户名、密码异常，授权类型不支持的异常进行处理
    private OAuthServerWebResponseExceptionTranslator translate;
    
    /**
     * 重写/oauth/check_token这个默认接口，用于校验令牌，返回的数据格式统一
     */
    @PostMapping(value = "/check_token")
    public ResultMsg<Map<String,?>> checkToken(@RequestParam("token") String value)  {
        Map<String, ?> map = checkTokenEndpoint.checkToken(value);
        return ResultMsg.resultSuccess(map);
    }
```



这种方式是不是很优雅？也很符合Spring Security的设计思想，AOP的方式还要对参数解析，重新包装

好了，关于测试的话自己搞一搞

## 总结

本篇文章介绍了认证服务中对token的返回格式自定义，总的来说还是比较简单的，有兴趣的也可以去网上找找关于AOP的方式。

