**大家好，我是不才陈某~**

认证、授权是实战项目中必不可少的部分，而Spring Security则将作为首选安全组件，因此陈某新开了 [《Spring Security 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2151150065472569352#wechat_redirect) 这个专栏，写一写从单体架构到**OAuth2**分布式架构的认证授权。

Spring security这里就不再过多介绍了，相信大家都用过，也都恐惧过，相比Shiro而言，Spring Security更加重量级，之前的SSM项目更多企业都是用的Shiro，但是Spring Boot出来之后，整合Spring Security更加方便了，用的企业也就多了。

今天陈某就来介绍一下在前后端分离的项目中如何使用Spring Security进行登录认证。文章的目录如下：

![](https://img.java-family.cn/Spring%20Security/19.png)

## 前后端分离认证的思路

前后端分离不同于传统的web服务，无法使用session，因此我们采用JWT这种无状态机制来生成token，大致的思路如下：

1. 客户端调用服务端登录接口，输入用户名、密码登录，登录成功返回两个**token**，如下：
   1. **accessToken**：客户端携带这个token访问服务端的资源
   2. **refreshToken**：刷新令牌，一旦accessToken过期了，客户端需要使用refreshToken重新获取一个accessToken。因此refreshToken的过期时间一般大于accessToken。
2. 客户请求头中携带**accessToken**访问服务端的资源，服务端对**accessToken**进行鉴定（验签、是否失效....），如果这个**accessToken**没有问题则放行。
3. **accessToken**一旦过期需要客户端携带**refreshToken**调用刷新令牌的接口重新获取一个新的**accessToken**。

## 项目搭建

陈某使用的是Spring Boot 框架，演示项目新建了两个模块，分别是`common-base`、`security-authentication-jwt`。

**1、common-base模块**

这是一个抽象出来的公共模块，这个模块主要放一些公用的类，目录如下：

![](https://img.java-family.cn/Spring%20Security/36.png)

**2、security-authentication-jwt模块**

一些需要定制的类，比如security的全局配置类、Jwt登录过滤器的配置类，目录如下：

![](https://img.java-family.cn/Spring%20Security/37.png)

**3、五张表**

权限设计根据业务的需求往往有不同的设计，陈某用的**RBAC**规范，主要涉及到五张表，分别是**用户表**、**角色表**、**权限表**、**用户<->角色表**、**角色<->权限表**，如下图：

![](https://img.java-family.cn/Spring%20Security/40.png)

> 上述几张表的SQL会放在案例源码中（这几张表字段为了省事，设计的并不全，自己根据业务逐步拓展即可）

## 登录认证过滤器

登录接口的逻辑写法有很多种，今天陈某介绍一种使用过滤器的定义的登录接口。

Spring Security默认的表单登录认证的过滤器是`UsernamePasswordAuthenticationFilter`，这个过滤器并不适用于前后端分离的架构，因此我们需要自定义一个过滤器。

逻辑很简单，参照`UsernamePasswordAuthenticationFilter`这个过滤器改造一下，代码如下：

![](https://img.java-family.cn/Spring%20Security/2.png)

## 认证成功处理器AuthenticationSuccessHandler

上述的过滤器接口一旦认证成功，则会调用**AuthenticationSuccessHandler**进行处理，因此我们可以自定义一个认证成功处理器进行自己的业务处理，代码如下：

![](https://img.java-family.cn/Spring%20Security/3.png)

陈某仅仅返回了**accessToken**、**refreshToken**，其他的业务逻辑处理自己完善。

## 认证失败处理器AuthenticationFailureHandler

同样的，一旦登录失败，比如用户名或者密码错误等等，则会调用**AuthenticationFailureHandler**进行处理，因此我们需要自定义一个认证失败的处理器，其中根据异常信息返回特定的**JSON**数据给客户端，代码如下：

![](https://img.java-family.cn/Spring%20Security/4.png)

逻辑很简单，**AuthenticationException**有不同的实现类，根据异常的类型返回特定的提示信息即可。

## AuthenticationEntryPoint配置

**AuthenticationEntryPoint**这个接口当**用户未通过认证访问受保护的资源**时，将会调用其中的`commence()`方法进行处理，比如客户端携带的token被篡改，因此我们需要自定义一个**AuthenticationEntryPoint**返回特定的提示信息，代码如下：

![](https://img.java-family.cn/Spring%20Security/5.png)



## AccessDeniedHandler配置

**AccessDeniedHandler**这处理器当认证成功的用户访问受保护的资源，但是**权限不够**，则会进入这个处理器进行处理，我们可以实现这个处理器返回特定的提示信息给客户端，代码如下：

![](https://img.java-family.cn/Spring%20Security/6.png)



## UserDetailsService配置

**UserDetailsService**这个类是用来加载用户信息，包括**用户名**、**密码**、**权限**、**角色**集合....其中有一个方法如下：

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

在认证逻辑中Spring Security会调用这个方法根据客户端传入的username加载该用户的详细信息，这个方法需要完成的逻辑如下：

- 密码匹配
- 加载权限、角色集合

我们需要实现这个接口，从**数据库**加载用户信息，代码如下：

![](https://img.java-family.cn/Spring%20Security/38.png)

其中的**LoginService**是根据用户名从数据库中查询出密码、角色、权限，代码如下：

![](https://img.java-family.cn/Spring%20Security/39.png)

**UserDetails**这个也是个接口，其中定义了几种方法，都是围绕着**用户名**、**密码**、**权限+角色集合**这三个属性，因此我们可以实现这个类拓展这些字段，**SecurityUser**代码如下：

![](https://img.java-family.cn/Spring%20Security/8.png)



**拓展**：**UserDetailsService**这个类的实现一般涉及到**5**张表，分别是**用户表**、**角色表**、**权限表**、**用户<->角色对应关系表**、**角色<->权限对应关系表**，企业中的实现必须遵循**RBAC**设计规则。这个规则陈某后面会详细介绍。

## Token校验过滤器

客户端请求头携带了token，服务端肯定是需要针对每次请求解析、校验token，因此必须定义一个Token过滤器，这个过滤器的主要逻辑如下：

- 从请求头中获取**accessToken**
- 对**accessToken**解析、验签、校验过期时间
- 校验成功，将authentication存入ThreadLocal中，这样方便后续直接获取用户详细信息。

上面只是最基础的一些逻辑，实际开发中还有特定的处理，比如将用户的详细信息放入Request属性中、Redis缓存中，这样能够实现feign的令牌中继效果。

校验过滤器的代码如下：

![](https://img.java-family.cn/Spring%20Security/9.png)



## 刷新令牌接口

**accessToken**一旦过期，客户端必须携带着**refreshToken**重新获取令牌，传统web服务是放在cookie中，只需要服务端完成刷新，完全做到无感知令牌续期，但是前后端分离架构中必须由客户端拿着**refreshToken**调接口手动刷新。

代码如下：

![](https://img.java-family.cn/Spring%20Security/10.png)

主要逻辑很简单，如下：

- 校验**refreshToken**
- 重新生成**accessToken**、**refreshToken**返回给客户端。

> 注意：实际生产中**refreshToken**令牌的生成方式、加密算法可以和**accessToken**不同。

## 登录认证过滤器接口配置

上述定义了一个认证过滤器**JwtAuthenticationLoginFilter**，这个是用来登录的过滤器，但是并没有注入加入Spring Security的过滤器链中，需要定义配置，代码如下：

```java
/**
 * @author 公众号：码猿技术专栏
 * 登录过滤器的配置类
 */
@Configuration
public class JwtAuthenticationSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {

    /**
     * userDetailService
     */
    @Qualifier("jwtTokenUserDetailsService")
    @Autowired
    private UserDetailsService userDetailsService;

    /**
     * 登录成功处理器
     */
    @Autowired
    private LoginAuthenticationSuccessHandler loginAuthenticationSuccessHandler;

    /**
     * 登录失败处理器
     */
    @Autowired
    private LoginAuthenticationFailureHandler loginAuthenticationFailureHandler;

    /**
     * 加密
     */
    @Autowired
    private PasswordEncoder passwordEncoder;

    /**
     * 将登录接口的过滤器配置到过滤器链中
     * 1. 配置登录成功、失败处理器
     * 2. 配置自定义的userDetailService（从数据库中获取用户数据）
     * 3. 将自定义的过滤器配置到spring security的过滤器链中，配置在UsernamePasswordAuthenticationFilter之前
     * @param http
     */
    @Override
    public void configure(HttpSecurity http) {
        JwtAuthenticationLoginFilter filter = new JwtAuthenticationLoginFilter();
        filter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
        //认证成功处理器
        filter.setAuthenticationSuccessHandler(loginAuthenticationSuccessHandler);
        //认证失败处理器
        filter.setAuthenticationFailureHandler(loginAuthenticationFailureHandler);
        //直接使用DaoAuthenticationProvider
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        //设置userDetailService
        provider.setUserDetailsService(userDetailsService);
        //设置加密算法
        provider.setPasswordEncoder(passwordEncoder);
        http.authenticationProvider(provider);
        //将这个过滤器添加到UsernamePasswordAuthenticationFilter之前执行
        http.addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

所有的逻辑都在`public void configure(HttpSecurity http)`这个方法中，如下：

- 设置认证成功处理器**loginAuthenticationSuccessHandler**
- 设置认证失败处理器**loginAuthenticationFailureHandler**
- 设置userDetailService的实现类**JwtTokenUserDetailsService**
- 设置加密算法passwordEncoder
- 将**JwtAuthenticationLoginFilter**这个过滤器加入到过滤器链中，直接加入到**UsernamePasswordAuthenticationFilter**这个过滤器之前。



## Spring Security全局配置

上述仅仅配置了登录过滤器，还需要在全局配置类做一些配置，如下：

- 应用登录过滤器的配置
- 将登录接口、令牌刷新接口放行，不需要拦截
- 配置**AuthenticationEntryPoint**、**AccessDeniedHandler**
- 禁用session，前后端分离+JWT方式不需要session
- 将token校验过滤器**TokenAuthenticationFilter**添加到过滤器链中，放在**UsernamePasswordAuthenticationFilter**之前。

完整配置如下：

```java
/**
 * @author 公众号：码猿技术专栏
 * @EnableGlobalMethodSecurity 开启权限校验的注解
 */
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true,securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private JwtAuthenticationSecurityConfig jwtAuthenticationSecurityConfig;
    @Autowired
    private EntryPointUnauthorizedHandler entryPointUnauthorizedHandler;
    @Autowired
    private RequestAccessDeniedHandler requestAccessDeniedHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                //禁用表单登录，前后端分离用不上
                .disable()
                //应用登录过滤器的配置，配置分离
                .apply(jwtAuthenticationSecurityConfig)

                .and()
                // 设置URL的授权
                .authorizeRequests()
                // 这里需要将登录页面放行,permitAll()表示不再拦截，/login 登录的url，/refreshToken刷新token的url
                //TODO 此处正常项目中放行的url还有很多，比如swagger相关的url，druid的后台url，一些静态资源
                .antMatchers(   "/login","/refreshToken")
                .permitAll()
                //hasRole()表示需要指定的角色才能访问资源
                .antMatchers("/hello").hasRole("ADMIN")
                // anyRequest() 所有请求   authenticated() 必须被认证
                .anyRequest()
                .authenticated()

                //处理异常情况：认证失败和权限不足
                .and()
                .exceptionHandling()
                //认证未通过，不允许访问异常处理器
                .authenticationEntryPoint(entryPointUnauthorizedHandler)
                //认证通过，但是没权限处理器
                .accessDeniedHandler(requestAccessDeniedHandler)

                .and()
                //禁用session，JWT校验不需要session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)

                .and()
                //将TOKEN校验过滤器配置到过滤器链中，否则不生效，放到UsernamePasswordAuthenticationFilter之前
                .addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class)
                // 关闭csrf
                .csrf().disable();
    }

    // 自定义的Jwt Token校验过滤器
    @Bean
    public TokenAuthenticationFilter authenticationTokenFilterBean()  {
        return new TokenAuthenticationFilter();
    }

    /**
     * 加密算法
     * @return
     */
    @Bean
    public PasswordEncoder getPasswordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
```

注释的很详细了，有不理解的认真看一下。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 测试

1、首先测试登录接口，postman访问http://localhost:2001/security-jwt/login，如下：

![](https://img.java-family.cn/Spring%20Security/41.png)

可以看到，成功返回了两个token。

2、请求头不携带token，直接请求http://localhost:2001/security-jwt/hello，如下：

![](https://img.java-family.cn/Spring%20Security/12.png)

可以看到，直接进入了**EntryPointUnauthorizedHandler**这个处理器。

3、携带token访问http://localhost:2001/security-jwt/hello，如下：

![](https://img.java-family.cn/Spring%20Security/13.png)

成功访问，token是有效的。

4、刷新令牌接口测试，携带一个过期的令牌访问如下：

![](https://img.java-family.cn/Spring%20Security/14.png)



5、刷新令牌接口测试，携带未过期的令牌测试，如下：

![](https://img.java-family.cn/Spring%20Security/15.png)

可以看到，成功返回了两个新的令牌。

## 源码追踪

以上一系列的配置完全是参照**UsernamePasswordAuthenticationFilter**这个过滤器，这个是web服务表单登录的方式。

Spring Security的原理就是一系列的过滤器组成，登录流程也是一样，起初在`org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter#doFilter()`方法，进行认证匹配，如下：

![](https://img.java-family.cn/Spring%20Security/17.png)

`attemptAuthentication()`这个方法主要作用就是获取客户端传递的username、password，封装成`UsernamePasswordAuthenticationToken`交给`ProviderManager`的进行认证，源码如下：

![](https://img.java-family.cn/Spring%20Security/16.png)

ProviderManager主要流程是调用抽象类`AbstractUserDetailsAuthenticationProvider#authenticate()`方法，如下图：

![](https://img.java-family.cn/Spring%20Security/18.png)

`retrieveUser()`方法就是调用userDetailService查询用户信息。然后认证，一旦认证成功或者失败，则会调用对应的失败、成功处理器进行处理。

## 总结

Spring Security虽然比较重，但是真的好用，尤其是实现Oauth2.0规范，非常简单方便。

> 案例源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9529** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！





