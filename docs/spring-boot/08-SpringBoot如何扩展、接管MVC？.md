
## 前言
自从用了Spring Boot是否有一个感觉，以前MVC的配置都很少用到了，比如视图解析器，拦截器，过滤器等等，这也正是Spring Boot好处之一。

但是往往Spring Boot提供默认的配置不一定适合实际的需求，因此需要能够定制MVC的相关功能，这篇文章就介绍一下如何扩展和全面接管MVC。

## Spring Boot 版本
本文基于的Spring Boot的版本是`2.3.4.RELEASE`。

## 如何扩展MVC？
在这里需要声明一个前提：**配置类上没有标注`@EnableWebMvc`并且没有任何一个配置类继承了`WebMvcConfigurationSupport`**。至于具体原因，下文会详细解释。

**扩展MVC其实很简单，只需要以下步骤**：
1. 创建一个MVC的配置类，并且标注`@Configuration`注解。
2. 实现`WebMvcConfigurer`这个接口，并且实现需要的方法。

`WebMvcConfigurer`这个接口中定义了MVC相关的各种组件，比如拦截器，视图解析器等等的定制方法，需要定制什么功能，只需要实现即可。

在Spring Boot之前的版本还可以继承一个抽象类`WebMvcConfigurerAdapter`，不过在`2.3.4.RELEASE`这个版本中被废弃了，如下：
```java
@Deprecated
public abstract class WebMvcConfigurerAdapter implements WebMvcConfigurer {}
```

**举个栗子**：现在要添加一个拦截器，使其在Spring Boot中生效，此时就可以在MVC的配置类重写`addInterceptors()`方法，如下：
```java
/**
 * MVC扩展的配置类，实现WebMvcConfigurer接口
 */
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private RepeatSubmitInterceptor repeatSubmitInterceptor;

    /**
     * 重写addInterceptors方法，注入自定义的拦截器
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(repeatSubmitInterceptor).excludePathPatterns("/error");
    }
}
```

操作很简单，除了拦截器，还可以定制视图解析，资源映射处理器等等相关的功能，和Spring MVC很类似，只不过Spring MVC是在`XML`文件中配置，Spring Boot是在配置类中配置而已。


## 什么都不配置为什么依然能运行MVC相关的功能？

早期的SSM架构中想要搭建一个MVC其实挺复杂的，需要配置视图解析器，资源映射处理器，`DispatcherServlet`等等才能正常运行，但是为什么Spring Boot仅仅是添加一个`WEB`模块依赖即能正常运行呢？依赖如下：
```java
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency> 
```
其实这已经涉及到了Spring Boot高级的知识点了，在这里就简单的说一下，Spring Boot的每一个`starter`都会有一个自动配置类，什么是自动配置类呢？**自动配置类就是在Spring Boot项目启动的时候会自动加载的类，能够在启动期间就配置一些默认的配置**。`WEB`模块的自动配置类是`WebMvcAutoConfiguration`。

`WebMvcAutoConfiguration`这个配置类中还含有如下一个子配置类`WebMvcAutoConfigurationAdapter`，如下：
```java
@Configuration(proxyBeanMethods = false)
	@Import(EnableWebMvcConfiguration.class)
	@EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
	@Order(0)
	public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {}
```

`WebMvcAutoConfigurationAdapter`这个子配置类实现了`WebMvcConfigurer`这个接口，这个正是MVC扩展接口，这个就很清楚了。**自动配置类是在项目启动的时候就加载的，因此Spring Boot会在项目启动时加载`WebMvcAutoConfigurationAdapter`这个MVC扩展配置类，提前完成一些默认的配置（比如内置了默认的视图解析器，资源映射处理器等等），这也就是为什么没有配置什么MVC相关的东西依然能够运行**。


## 如何全面接管MVC？【不推荐】

全面接管MVC是什么意思呢？全面接管的意思就是不需要Spring Boot自动配置，而是全部使用自定义的配置。

**全面接管MVC其实很简单，只需要在配置类上添加一个`@EnableWebMvc`注解即可**。还是添加拦截器，例子如下：
```java
/**
 * @EnableWebMvc：全面接管MVC，导致自动配置类失效
 */
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {
    @Autowired
    private RepeatSubmitInterceptor repeatSubmitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //添加拦截器
        registry.addInterceptor(repeatSubmitInterceptor).excludePathPatterns("/error");
    }
}
```

一个注解就能全面接口MVC，是不是很爽，不过，不建议使用。

## 为什么@EnableWebMvc一个注解就能够全面接管MVC？

what？？？为什么呢？上面刚说过自动配置类`WebMvcAutoConfiguration`会在项目启动期间加载一些默认的配置，这会怎么添加一个`@EnableWebMvc`注解就不行了呢？
![](https://img.java-family.cn/Spring%20Boot%E7%AC%AC%E5%85%AB%E5%BC%B9%EF%BC%8C%E5%A6%82%E4%BD%95%E6%89%A9%E5%B1%95%E5%85%A8%E9%9D%A2%E6%8E%A5%E5%8F%A3MVC/1.jpg)

其实很简单，`@EnableWebMvc`源码如下：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

其实重要的就是这个`@Import(DelegatingWebMvcConfiguration.class)`注解了，Spring中的注解，快速导入一个配置类`DelegatingWebMvcConfiguration`，源码如下：
```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {}
```

**明白了，`@EnableWebMvc`这个注解实际上就是导入了一个`WebMvcConfigurationSupport`子类型的配置类而已**。

而WEB模块的自动配置类有这么一行注解`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`，源码如下：
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
```

这个注解`@ConditionalOnMissingBean`什么意思呢？简单的说就是IOC容器中没有指定的`Bean`这个配置才会生效。

**一切都已经揭晓了，`@EnableWebMvc`导入了一个`WebMvcConfigurationSupport`类型的配置类，导致了自动配置类`WebMvcAutoConfiguration`标注的`@@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`判断为`false`了，从而自动配置类失效了。**

## Spring Boot相关资料
前期有很多的小伙伴私信我，觉得看文章太枯燥了，有些东西也不能理解的透彻，有没有好的视频课程分享，前几天特意回家找了找资源，总算找到了适合入门学习的完整视频教程，从Spring Boot初级入门到高级整合，讲解的非常全面，一些目录如下：

![](https://img.java-family.cn/Spring%20Boot%E7%AC%AC%E5%85%AB%E5%BC%B9%EF%BC%8C%E5%A6%82%E4%BD%95%E6%89%A9%E5%B1%95%E5%85%A8%E9%9D%A2%E6%8E%A5%E5%8F%A3MVC/3.png)

这些资料全部免费提供，我的文章也是尽量跟着视频大纲匹配，希望小伙伴能够系统完整的学习Spring Boot。**公众号【码猿技术专栏】回复关键词`Spring Boot初级`和`Spring Boot高级`分别获取初级和高级的视频教程。**

## 总结

扩展和全面接管MVC都很简单，但是不推荐全面接管MVC，一旦全面接管了，WEb模块的这个`starter`将没有任何意义，一些全局配置文件中与MVC相关的配置也将会失效。







