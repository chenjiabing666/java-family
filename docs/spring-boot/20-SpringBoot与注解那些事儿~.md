

## 前言
注解相信大家都用过，尤其是`Spring Boot` 这个框架，比如`@Controller`。

这篇文章就来介绍下`Spring Boot` 中如何自定义一个注解，顺带介绍一下`Spring Boot` 与 `AOP`如何整合。

## 什么是AOP？
`AOP`即是面向切面，是`Spring`的核心功能之一，主要的目的即是针对业务处理过程中的横向拓展，以达到低耦合的效果。

举个栗子，项目中有记录操作日志的需求、或者流程变更是记录变更履历，无非就是插表操作，很简单的一个`save`操作，都是一些记录日志或者其他辅助性的代码。一遍又一遍的重写和调用。不仅浪费了时间，又将项目变得更加的冗余，实在得不偿失。

此时`AOP`的就该出场了，能够在不改变原逻辑的基础上实现相关功能。

## AOP的相关概念（面试常客）
要理解`Spring Boot`整合`Aop`的实现，就必须先对面向切面实现的一些`Aop`的概念有所了解，不然也是云里雾里。

**切面（Aspect）**：一个关注点的模块化。以注解`@Aspect`的形式放在类上方，声明一个切面。

**连接点（Joinpoint）**：在程序执行过程中某个特定的点，比如某方法调用的时候或者处理异常的时候都可以是连接点。

**通知（Advice）**：通知增强，需要完成的工作叫做通知，就是你写的业务逻辑中需要比如事务、日志等先定义好，然后需要的地方再去用。增强包括如下五个方面：
1. `@Before`：在切点之前执行
2. `@After`：在切点方法之后执行
3. `@AfterReturning`：切点方法返回后执行
4. `@AfterThrowing`：切点方法抛异常执行
5. `@Around`：属于环绕增强，能控制切点执行前，执行后，用这个注解后，程序抛异常，会影响`@AfterThrowing`这个注解。

**切点（Pointcut）**：其实就是筛选出的连接点，匹配连接点的断言，一个类中的所有方法都是连接点，但又不全需要，会筛选出某些作为连接点做为切点。

**引入（Introduction）**：在不改变一个现有类代码的情况下，为该类添加属性和方法,可以在无需修改现有类的前提下，让它们具有新的行为和状态。其实就是把切面（也就是新方法属性：通知定义的）用到目标类中去。

**目标对象（Target Object）**：被一个或者多个切面所通知的对象。也被称做被通知（`adviced`）对象。既然`Spring AOP`是通过运行时代理实现的，这个对象永远是一个被代理（`proxied`）对象。

**AOP代理（AOP Proxy）**：`AOP`框架创建的对象，用来实现切面契约（例如通知方法执行等等）。在`Spring`中，`AOP`代理可以是`JDK`动态代理或者`CGLIB`代理。

**织入（Weaving）**：把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时（例如使用`AspectJ`编译器），类加载时和运行时完成。`Spring`和其他纯`Java AOP`框架一样，在运行时完成织入。


## Spring Boot 如何整合AOP自定义一个注解？
在实际开发中对于横向公共的逻辑需要抽取出来，这时候就需要使用`AOP`，比如日志的记录、权限的验证等等，这些功能都可以用注解轻松的完成。

下面介绍如何在`Spring Boot`使用`AOP`定义一个注解。

### 添加依赖starter
`AOP`整合`Spring Boot`有一个`starter`，只需要添加依赖即可，如下：
```xml
<!--springboot集成Aop-->
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-aop</artifactId>
  </dependency>
```

### 开启AOP
在配置类上标注`@EnableAspectJAutoProxy`注解即可开启`AOP`，这个注解有什么用呢，源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {}
```

最重要的是如下一行代码：
```java
@Import(AspectJAutoProxyRegistrar.class)
```

`@Import`这个注解很熟悉了吧，快速注入一个类，这里是注入一个`AnnotationAwareAspectJAutoProxyCreator`。


### 自定义一个注解
就以日志处理为例子，定义一个日志处理的注解，如下：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SysLog {
    String value() default "";
}
```

### 定义一个切面
一个切面的满足条件如下：
1. 类上标注了`@Aspect`注解
2. 注入到IOC容器中，比如`@Component`注解

定义的日志切面如下：
```java
@Component
@Aspect
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SysLogAspect {
}
```

`@Order`指定了切面执行的优先级，假如有多个切面，肯定是要有先后的执行顺序，这样才能保证逻辑性。


### 定义切点表达式
这里需要拦截的肯定是`@SysLog`这个注解，只要方法上标注了该注解都将会被拦截，表达式如下：
```java
@Pointcut("@annotation(com.example.annotation_demo.annotation.SysLog)")
public void pointCut() {}
```

### 添加通知方法

既然是日志记录，肯定是在方法执行前，执行后都需要记录，因此需要定义一个环绕通知，如下：
```java
  @Around("pointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        //逻辑开始时间
        long beginTime = System.currentTimeMillis();

        //执行方法
        Object result = point.proceed();

        //todo，保存日志，自己完善
        saveLog(point,beginTime);

        return result;
    }
```

### 测试
以上配置完成后即可使用，只需要在需要的方法上标注`@SysLog`注解即可，如下：
```java
@SysLog
@PostMapping("/add")
public String add(){
  return "";
}
```

## 使用拦截器如何自定义注解？
使用`AOP`自定义的注解在每个方法上都会被拦截验证，首先效率上就不高。

然而拦截器是在每个`Controller`方法执行之前进行拦截，其他的方法都不会生效，比如`service`方法。

比如权限的验证、防止瞬间重复点击等等需求就适合使用拦截器自定义的注解。

### 自定义一个注解
就以防止瞬间重复点击的例子来创建一个注解，如下：
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RepeatSubmit {
    /**
     * 默认失效时间5秒
     */
    long seconds() default 5;
}
```

### 自定义拦截器
需要在请求执行之前完成验证，逻辑很简单，就是判断方法上有没有标注`@RepeatSubmit`注解，代码如下：
```java
/**
 * description:重复提交注解的拦截器
 */
@Component
public class RepeatSubmitInterceptor implements HandlerInterceptor {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod){
            //只拦截标注了@RepeatSubmit该注解
            HandlerMethod handlerMethod=(HandlerMethod)handler;
            //获取controller方法上标注的注解
            RepeatSubmit repeatSubmit = AnnotationUtils.findAnnotation(handlerMethod.getMethod(),RepeatSubmit.class);
            //没有限制重复提交，直接跳过
            if (Objects.isNull(repeatSubmit))
                return true;
            //todo 一个值，标志这个请求的唯一性，比如IP+userId+uri+请求参数
            String flag="";
            //存在即返回false，不存在即返回true
            Boolean ifAbsent = stringRedisTemplate.opsForValue().setIfAbsent(flag, "", repeatSubmit.seconds(), TimeUnit.SECONDS);
            if (ifAbsent!=null&&!ifAbsent)
                //todo: 此处抛出异常，需要在全局异常解析器中捕获
                throw new RepeatSubmitException();
        }
        return true;
    }
}
```

### 注入的拦截器
将上述自定义的拦截器注入到`Sprign Boot`中，这里不再演示了，前面教程有介绍过，请看：[Spring Boot 第六弹，拦截器如何配置，看这儿~](https://mp.weixin.qq.com/s/PDg-0sZ_FUcS8ERenBWPYg)。

### 测试
在需要拦截方法上添加`@RepeatSubmit`注解即可，如下：
```java
    @RepeatSubmit
    @GetMapping("/add")
    public String add(){
        return "";
    }
```

## 内部调用导致AOP注解失效
这个问题在事务中也是经常被忽略的问题，网上很多人说是`AOP`的`Bug`，其实在我看来这真不是一个`BUG`，并且也是有办法解决的。

先来看一下失效的案例，如下：
```java
public class ArticleServiceImpl{
  @SysLog
  public void A(){
    ......
  }
  
  
  public void B(){
    this.A();
  }
}
```

在上述的代码中，如果执行方法`B`，则`@SysLog`注解将会失效。

### 失效的原因
`AOP`使用的是动态代理的机制，它会给类生成一个代理类，事务的相关操作都在代理类上完成。内部方式使用`this`调用方式时，使用的是实例调用，并没有通过代理类调用方法，所以会导致事务失效。

### 解决方法
其实解决方法有很多，下面将会一一介绍。

#### 1. 引入自身的Bean

在类内部通过`@Autowired`将本身`bean`引入，然后通过调用自身`bean`，从而实现使用`AOP`代理操作。代码如下：
```java
public class ArticleServiceImpl{
  /**
  * 注入自身的Bean
  */
  @Autowired
  private ArticleService articleService;
  
  @SysLog
  public void A(){
    ......
  }
  
  public void B(){
    articleService.A();
  }
}
```

#### 2. 通过ApplicationContext引入bean
通过`ApplicationContext`获取`bean`，通过`bean`调用内部方法，就使用了`bean`的代理类。

需要先创建一个`ApplicationContext`的工具类获取`ApplicationContext`，然后才能调用`getBean()`方法，代码如下：
```java
public class ArticleServiceImpl{
  
  @SysLog
  public void A(){
    ......
  }
  
  public void B(){
    ApplicationContextUtils.getApplicationContext().getBean(ArticleService.class).A();
  }
}
```

#### 3. 通过AopContext获取当前类的代理类
此种方法需要设置`@EnableAspectJAutoProxy`中的`exposeProxy`为`true`。

使用`AopContext`获取当前的代理对象，代码如下：
```java
public class ArticleServiceImpl{
  
  @SysLog
  public void A(){
    ......
  }
  
  public void B(){
    ((ArticleService)AopContext.currentProxy()).A();
  }
}
```

## 总结
这篇文章介绍了`AOP`的相关概念、`AOP`实现自定义注解以及拦截器实现自定义注解，都是日常开发中必备的知识点，希望这篇文章对各位有所帮助。

> 源码已经上传，回复关键词`AOP注解`获取。

