**大家好，我是不才陈某~**

这是[《Spring Security 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2151150065472569352#wechat_redirect)的第**13**篇文章，往期文章如下：

- [实战！Spring Boot Security+JWT前后端分离架构登录认证！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502546&idx=1&sn=bfb6fd9d96d8c5bf107a4981ba5e1547&chksm=fcf7151fcb809c09b7ae29de8c0af0d00976539a46ee5f9bf583a6a7b196ea82f26ce98fd982&scene=178&cur_album_id=2151150065472569352#rd)
- [妹子始终没搞懂OAuth2.0，今天整合Spring Cloud Security 一次说明白！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502682&idx=1&sn=52a15b623ab6135c134b8262bd605946&chksm=fcf71497cb809d81f1d2dbce76b3e00170f085306b2a2a67a807a6d9e2cf03bf1de3b8f203a2&scene=178&cur_album_id=2151150065472569352#rd)
- [OAuth2.0实战！使用JWT令牌认证！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502801&idx=1&sn=56b1af09bfa25d5e44193a7d75dfa623&chksm=fcf7141ccb809d0a1b0b2d7f6d9893c7d3e560dd8996296276f0274d2578236ee87e9124810d&scene=178&cur_album_id=2151150065472569352#rd)
- [OAuth2.0实战！玩转认证、资源服务异常自定义这些骚操作！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502905&idx=1&sn=32ba3ae4e0a4097d238f64719c88b7f7&chksm=fcf713f4cb809ae2ccb706b8e9f8184739d3c7b97d388467bfe9ff31a8c15768b6de05054f08&scene=178&cur_album_id=2151150065472569352#rd)
- [实战干货！Spring Cloud Gateway 整合 OAuth2.0 实现分布式统一认证授权！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247503249&idx=1&sn=b33ae3ff70a08b17ee0779d6ccb30b53&chksm=fcf7125ccb809b4aa4985da09e620e06c606754e6a72681c93dcc88bdc9aa7ba0cb64f52dbc3&scene=178&cur_album_id=2151150065472569352#rd)
- [实战！退出登录时如何借助外力使JWT令牌失效？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247504322&idx=1&sn=4b0a2488a4edcb025d0694604e86f840&chksm=fcf70e0fcb808719b98a65891bc08e9490db09f07debd4521052978a319ab052f8a72b93c1a7&scene=178&cur_album_id=2151150065472569352#rd)
- [实战！Spring Cloud Gateway集成 RBAC 权限模型实现动态权限控制！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247504442&idx=1&sn=48c1dd73c038e3d936db4e5134f7bbc2&chksm=fcf70df7cb8084e1556cac092fdb68ffd6503cbb287485ef76e9941611d134258c38d03e890a&scene=178&cur_album_id=2151150065472569352#rd)
- [实战！openFeign如何实现全链路JWT令牌信息不丢失？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247504759&idx=1&sn=e50d5b44eb64debf43c6d644f55c68b5&chksm=fcf70cbacb8085aca5cd88688973ed45cd8bd9a4642ae97727f3684b431f80316e5c073d6946&scene=178&cur_album_id=2151150065472569352#rd)
- [3 个注解，优雅的实现微服务鉴权](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247512286&idx=1&sn=4ca3339cc0428c72a7885957d5ee9241&chksm=fcf76f13cb80e605f878498bae159cfe2b55f03334b32dd77bac46d2fc3e2c0d2e5cafbd7d7c&scene=178&cur_album_id=2151150065472569352#rd)
- [微服务中使用阿里开源的TTL，优雅的实现身份信息的线程间复用](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247512365&idx=1&sn=f847a72fecda9852ad23879e78e07af2&chksm=fcf76ee0cb80e7f6df3c15868f89a7722db5152625b2e60c2ba78f4bc14112fbc45280e52fac&scene=178&cur_album_id=2151150065472569352#rd)
- [一个接口优雅的实现 Spring Cloud OAuth2 自定义token返回格式](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247514149&idx=1&sn=48e6952620d9fdb2363aa6c86af0e2f2&chksm=fcf767e8cb80eefeefdc23e35600cf296af25c07a1da77526489cc4fc58a19d3f53723f5a5f2&scene=178&cur_album_id=2151150065472569352#rd)
- [几行代码搞定 Spring Cloud OAuth2 授权码模式3个页面定制](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247514685&idx=1&sn=f886c3f8161c8a6e87278dbde9154018&chksm=fcf765f0cb80ece64b4c39184500246a5638b7fff9b9efa6d66ba517c3336e0a72d03f2da1d6&scene=178&cur_album_id=2151150065472569352#rd)

今天来聊一聊spring security中的一种经典认证模式HttpBasic，在5.x版本之前作为Spring Security默认认证模式，但是在5.x版本中被放弃了，默认的是form login认证模式



## HttpBasic模式的应用场景

HttpBasic登录验证模式是Spring Security实现登录验证最简单的一种方式，也可以说是最简陋的一种方式。

为什么是最简陋的？这种模式用来糊弄普通用户可以，但是稍微懂点技术的用户分分钟就可以将其破解，因为底层并未做任何的安全的设置，仅仅是将`用户名:密码`做了简单的base64加密传递给服务端，base64又是一种可逆的算法。

因此 HttpBasic 的应用场景非常少，对于不重要的数据，用户比较少但是又想设置一重障碍的时候就可以考虑使用这种



## 整合Spring Security 搞一把

虽然这种认证模式不太重要，但是还是要了解，对于后面的学习至关重要，下面搭建一个项目演示一下



### 1. 添加maven依赖

直接添加Spring Security的依赖，如下：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



### 2. Spring Security 添加配置

由于陈某使用的是Spring Boot 2.x版本，此时的Spring Security 是`5.x`版本，默认的认证方式是form表单认证，因此需要配置一下HttpBasic认证模式，代码如下：

```java
/**
 * @author 公众号：码猿技术专栏
 * @url: www.java-family.cn
 * @description Spring Security的配置类
 */
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic()//开启httpbasic认证
                .and()
                .authorizeRequests()
                .anyRequest()
                .authenticated();//所有请求都需要登录认证才能访问
    }
}
```

启动项目，在项目后台有这样的一串日志打印，冒号后面的就是默认密码。

```java
Using generated security password: 00af0f93-7103-4c8a-87a4-23a050a4285c
```

我们可以通过浏览器进行登录验证，默认的用户名是**user**.（下面的登录框不是我们开发的，是HttpBasic模式自带的）

![](https://img.java-family.cn/20220704204930.png)

当然我们也可以通过application.yml指定配置用户名密码，配置如下：

```yaml
spring:
    security:
      user:
        name: admin
        password: admin
```



## HttpBasic的原理

整个流程如下图：

![](https://img.java-family.cn/20220704210632.png)

1. 首先，HttpBasic模式要求传输的用户名密码使用Base64模式进行加密。如果用户名是 `admin` ，密码是admin，则将字符串`admin:admin`使用Base64编码算法加密。加密结果可能是：YWtaW46YWRtaW4=。
2. 然后，在Http请求中使用Authorization作为一个Header，`Basic YWtaW46YWRtaW4=`作为Header的值，发送给服务端。（注意这里使用Basic+空格+加密串）
3. 服务器在收到这样的请求时，到达BasicAuthenticationFilter过滤器，将提取“ Authorization”的Header值，并使用用于验证用户身份的相同算法Base64进行解码。
4. 解码结果与登录验证的用户名密码匹配，匹配成功则可以继续过滤器后续的访问。

所以，HttpBasic模式真的是非常简单又简陋的验证模式，Base64的加密算法是可逆的，你知道上面的原理，分分钟就破解掉。我们完全可以使用PostMan工具，发送Http请求进行登录验证。

![](https://img.java-family.cn/20220704211301.png)

整个流程都在`BasicAuthenticationFilter#doFilterInternal()`这个方法中，有兴趣的可以去看看。
