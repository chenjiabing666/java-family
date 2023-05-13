


## 前言
为什么`Spring Boot`这么火？因为便捷，开箱即用，但是你思考过为什么会这么便捷吗？传统的SSM架构配置文件至少要写半天，而使用`Spring Boot`之后只需要引入一个`starter`之后就能直接使用，why？？？

原因很简单，每个`starter`内部做了工作，比如`Mybatis`的启动器默认内置了可用的`SqlSessionFactory`。

至于如何内置的？`Spring Boot` 又是如何使其生效的？这篇文章就从源码角度介绍一下`Spring Boot`的自动配置原理。

## 源码版本
作者`Spring Boot`是基于`2.4.0`。每个版本有些变化，读者尽量和我保持一致，以防源码有些出入。

## @SpringBootApplication干了什么？
这么说吧，这个注解什么也没做，废物，活都交给属下做了，源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```
上方标注了三个重要的注解，如下：
1. `@SpringBootConfiguration`：其实就是`@Configuration`，因此主启动类可以当做配置类使用，比如注入`Bean`等。
2. `@EnableAutoConfiguration`：这个注解牛批了，名字就不一样，开启自动配置，哦，关键都在这了.....
3. `@ComponentScan`：包扫描注解。

经过以上的分析，最终定位了一个注解`@EnableAutoConfiguration`，顾名思义，肯定和自动配置有关，要重点分析下。

## @EnableAutoConfiguration干了什么？

想要知道做了什么肯定需要看源码，如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```
上方标注了两个重要的注解，如下：
1. `@AutoConfigurationPackage`：自动配置包注解，默认将主配置类(`@SpringBootApplication`)所在的包及其子包里面的所有组件扫描到`IOC容器`中。
2. `@Import`：该注解不必多说了，前面文章说过很多次了，这里是导入了`AutoConfigurationImportSelector`，用来注入自动配置类。

以上只是简单的分析了两个注解，下面将会从源码详细的介绍一下。

### @AutoConfigurationPackage
这个注解干了什么？这个需要看下源码，如下；
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {}
```

重要的还是`@Import`注解，导入了`AutoConfigurationPackages.Registrar`，这个类是干什么的？源码如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E5%8E%9F%E7%90%86/1.png)

其实就两个方法，但是的最重要的就是`registerBeanDefinitions`方法，但是这个方法不用看，肯定是注入`Bean`，这里的重点是注入哪些`Bean`，重点源码如下：
```java
//获取扫描的包
new PackageImports(metadata).getPackageNames().toArray(new String[0])
```

跟进代码，主要逻辑都在`#PackageImports.PackageImports()`这个构造方法中，源码解析如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E5%8E%9F%E7%90%86/2.png)

从上面源码分析可以知道，这里扫描的包名是由两部分组成，分别如下：
1. 从`@AutoConfigurationPackage`注解中的两个属性解析得来的包名。
2. 注解`AutoConfigurationPackage`所在的包名，即是`@SpringBootApplication`所在的包名。

> `@AutoConfigurationPackage`默认将主配置类(`@SpringBootApplication`)所在的包及其子包里面的所有组件扫描到IOC容器中。


### @Import(AutoConfigurationImportSelector.class) 
这个注解不用多说了，最重要的就是`AutoConfigurationImportSelector`，我们来看看它的继承关系，如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E5%8E%9F%E7%90%86/3.png)

这个类的继承关系还是挺简单的，实现了`Spring`中的`xxAware`注入一些必要的组件，但是最值得关心的是实现了一个`DeferredImportSelector`这个接口，这个接口扩展了`ImportSelector`，也改变了其运行的方式，这个在后面章节会介绍。

> **注意**：这个类会导致一个误区，平时看到`xxxSelector`已经有了反射弧了，肯定会在`selectImports()`方法上`DEBUG`，但是这个类压根就没执行该方法，我第一次看也有点怀疑人生了，原来它走的是`DeferredImportSelector`的接口方法。

其实该类真正实现逻辑的方法是`process()`方法，但是主要加载自动配置类的任务交给了`getAutoConfigurationEntry()`方法，具体的逻辑如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE%E5%8E%9F%E7%90%86/4.png)

上图的逻辑很简单，先从`spring.factories`文件中获取自动配置类，在去掉`@SpringBootApplication`中定义排除的自动配置类。

上图中的第`④`步就是从`META-INF/spring.factories`中加载自动配置类，代码很简单，在上一篇分析启动流程的时候也有很多组件是从`spring.facotries`文件中加载的，代码都类似。

在`springboot-autoconfigure`中的`spring.facotries`文件内置了很多自动配置类，如下：
```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
................
```

> 了解了`Spring Boot` 如何加载自动配置类，那么自定义一个自动配置类也是很简单了，后续章节教你如何定制自己的自动配置类，里面还是有很多门道的.....

## 总结
本文从源码角度分析了`Spring Boot`的自动配置是如何加载的，其实分析起来很简单，希望作者的这篇文章能帮助你更深层次的了解`Spring Boot`。














