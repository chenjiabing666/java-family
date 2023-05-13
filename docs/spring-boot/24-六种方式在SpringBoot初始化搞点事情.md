## 前言

在实际工作中总是需要在项目启动时做一些初始化的操作，比如初始化线程池、提前加载好加密证书.......

那么经典问题来了，这也是面试官经常会问到的一个问题：有哪些手段在Spring Boot 项目启动的时候做一些事情？

方法有很多种，下面介绍几种常见的方法。



## 1、监听容器刷新完成扩展点`ApplicationListener<ContextRefreshedEvent>`

ApplicationContext事件机制是观察者设计模式实现的，通过ApplicationEvent和ApplicationListener这两个接口实现ApplicationContext的事件机制。

Spring中一些内置的事件如下：

1. `ContextRefreshedEvent`：ApplicationContext 被初始化或刷新时，该事件被发布。这也可以在 ConfigurableApplicationContext接口中使用 refresh() 方法来发生。此处的初始化是指：所有的Bean被成功装载，后处理Bean被检测并激活，所有Singleton Bean 被预实例化，ApplicationContext容器已就绪可用。
2. `ContextStartedEvent`：当使用 ConfigurableApplicationContext （ApplicationContext子接口）接口中的 start() 方法启动 ApplicationContext 时，该事件被发布。你可以调查你的数据库，或者你可以在接受到这个事件后重启任何停止的应用程序。
3. `ContextStoppedEvent`：当使用 ConfigurableApplicationContext 接口中的 stop() 停止 ApplicationContext 时，发布这个事件。你可以在接受到这个事件后做必要的清理的工作。
4. `ContextClosedEvent`：当使用 ConfigurableApplicationContext 接口中的 close() 方法关闭 ApplicationContext 时，该事件被发布。一个已关闭的上下文到达生命周期末端；它不能被刷新或重启。
5. `RequestHandledEvent`：这是一个 web-specific 事件，告诉所有 bean HTTP 请求已经被服务。只能应用于使用DispatcherServlet的Web应用。在使用Spring作为前端的MVC控制器时，当Spring处理用户请求结束后，系统会自动触发该事件。

好了，了解上面这些内置事件后，我们可以监听`ContextRefreshedEvent`在Spring Boot 启动时完成一些操作，代码如下：

```java
@Component
public class TestApplicationListener implements ApplicationListener<ContextRefreshedEvent>{
    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        System.out.println(contextRefreshedEvent);
        System.out.println("TestApplicationListener............................");
    }
}
```

### 高级玩法

可以自定事件完成一些特定的需求，比如：邮件发送成功之后，做一些业务处理。

1. 自定义EmailEvent，代码如下：

```java
public class EmailEvent extends ApplicationEvent{
　　 private String address;
　　 private String text;
　　 public EmailEvent(Object source, String address, String text){
　　 super(source);
　　　　　 this.address = address;
　　　　　 this.text = text;
　　 }
　　 public EmailEvent(Object source) {
　　　　　super(source);
　　 }
　　 //......address和text的setter、getter
}
```

2. 自定义监听器，代码如下：

```java
public class EmailNotifier implements ApplicationListener{
　　 public void onApplicationEvent(ApplicationEvent event) {
　　　　　if (event instanceof EmailEvent) {
　　　　　　　 EmailEvent emailEvent = (EmailEvent)event;
　　　　　　　 System.out.println("邮件地址：" + emailEvent.getAddress());
　　　　　　　 System.our.println("邮件内容：" + emailEvent.getText());
　　　　　} else {
　　　　　　　 System.our.println("容器本身事件：" + event);
　　　　　}
　　 }
}
```

3. 发送邮件后，触发事件，代码如下：

```java
public class SpringTest {
　　 public static void main(String args[]){
　　　　　ApplicationContext context = new ClassPathXmlApplicationContext("bean.xml");
　　　　　//创建一个ApplicationEvent对象
　　　　　EmailEvent event = new EmailEvent("hello","abc@163.com","This is a test");
　　　　　//主动触发该事件
　　　　　context.publishEvent(event);
　　 }
}
```

## 2、`SpringBoot`的`CommandLineRunner`接口

当容器初始化完成之后会调用`CommandLineRunner`中的`run()`方法，同样能够达到容器启动之后完成一些事情。这种方式和ApplicationListener相比更加灵活，如下：

- 不同的`CommandLineRunner`实现可以通过`@Order()`指定执行顺序
- 可以接收从控制台输入的参数。

下面自定义一个实现类，代码如下：

```java
@Component
@Slf4j
public class CustomCommandLineRunner implements CommandLineRunner {

    /**
     * @param args 接收控制台传入的参数
     */
    @Override
    public void run(String... args) throws Exception {
        log.debug("从控制台接收参数>>>>"+ Arrays.asList(args));
    }
}
```

运行这个jar，命令如下：

```java
java -jar demo.jar aaa bbb ccc
```

以上命令中传入了三个参数，分别是`aaa`、`bbb`、`ccc`，这三个参数将会被`run()`方法接收到。如下图：

![](https://img.java-family.cn/spring%20Boot%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%90%9E%E7%82%B9%E4%BA%8B%E6%83%85/1.png)



### 源码分析

读过我的文章的铁粉都应该知道`CommandLineRunner`是如何执行的，原文：[头秃系列，二十三张图带你从源码分析Spring Boot 启动流程~](https://mp.weixin.qq.com/s/jlybHlukL1z_bSADXXiJ0g)

Spring Boot 加载上下文的入口在`org.springframework.context.ConfigurableApplicationContext()`这个方法中，如下图：

![](https://img.java-family.cn/spring%20Boot%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%90%9E%E7%82%B9%E4%BA%8B%E6%83%85/2.png)

调用CommandLineRunner在`callRunners(context, applicationArguments);`这个方法中执行，源码如下图：

![](https://img.java-family.cn/spring%20Boot%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%90%9E%E7%82%B9%E4%BA%8B%E6%83%85/3.png)



## 3、`SpringBoot`的`ApplicationRunner`接口

`ApplicationRunner`和`CommandLineRunner`都是Spring Boot 提供的，相对于`CommandLineRunner`来说对于控制台传入的参数封装更好一些，可以通过键值对来获取指定的参数，比如`--version=2.1.0`。

此时运行这个jar命令如下：

```java
java -jar demo.jar --version=2.1.0 aaa bbb ccc
```

以上命令传入了四个参数，一个键值对`version=2.1.0`，另外三个是分别是`aaa`、`bbb`、`ccc`。

同样可以通过`@Order()`指定优先级，如下代码：

```java
@Component
@Slf4j
public class CustomApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        log.debug("控制台接收的参数：{},{},{}",args.getOptionNames(),args.getNonOptionArgs(),args.getSourceArgs());
    }
}
```

通过以上命令运行，结果如下图：

![](https://img.java-family.cn/spring%20Boot%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%90%9E%E7%82%B9%E4%BA%8B%E6%83%85/4.png)



### 源码分析

和`CommandLineRunner`一样，同样在`callRunners()`这个方法中执行，源码如下图：

![](https://img.java-family.cn/spring%20Boot%20%E5%88%9D%E5%A7%8B%E5%8C%96%E6%90%9E%E7%82%B9%E4%BA%8B%E6%83%85/5.png)



## 4、`@PostConstruct`注解

前三种针对的是容器的初始化完成之后做的一些事情，`@PostConstruct`这个注解是针对`Bean`的初始化完成之后做一些事情，比如注册一些监听器...

`@PostConstruct`注解一般放在Bean的方法上，一旦Bean初始化完成之后，将会调用这个方法，代码如下：

```java
@Component
@Slf4j
public class SimpleExampleBean {

    @PostConstruct
    public void init(){
        log.debug("Bean初始化完成，调用...........");
    }
}
```

## 5、@Bean注解中指定初始化方法

这种方式和`@PostConstruct`比较类似，同样是指定一个方法在Bean初始化完成之后调用。

新建一个Bean，代码如下：

```java
@Slf4j
public class SimpleExampleBean {

    public void init(){
        log.debug("Bean初始化完成，调用...........");
    }
}
```

在配置类中通过`@Bean`实例化这个Bean，不过`@Bean`中的`initMethod`这个属性需要指定初始化之后需要执行的方法，如下：

```java
@Bean(initMethod = "init")
    public SimpleExampleBean simpleExampleBean(){
        return new SimpleExampleBean();
    }
```



## 6、 `InitializingBean`接口

`InitializingBean`的用法基本上与`@PostConstruct`一致，只不过相应的`Bean`需要实现`afterPropertiesSet`方法，代码如下：

```java
@Slf4j
@Component
public class SimpleExampleBean implements InitializingBean {

    @Override
    public void afterPropertiesSet()  {
        log.debug("Bean初始化完成，调用...........");
    }
}
```



## 总结

实现方案有很多，作者只是总结了常用的六种，学会的点个赞。
