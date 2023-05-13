
最近公司面试新人，无意中听到这样一道面试题：**Spring Boot 定时任务如何开启多线程？**

本来一道简单的面试题，却有不少人不了解。

今天这篇文章就来介绍一下Spring Boot 中 是如何开启多线程定时任务。

## 为什么Spring Boot 定时任务是单线程的？

想要解释为什么，一定要从源码入手，直接从`@EnableScheduling`这个注解入手，找到了这个**ScheduledTaskRegistrar**类，其中有一段代码如下：

```java
protected void scheduleTasks() {
		if (this.taskScheduler == null) {
			this.localExecutor = Executors.newSingleThreadScheduledExecutor();
			this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
		}
}
```

如果**taskScheduler**为**null**，则创建单线程的线程池：**Executors.newSingleThreadScheduledExecutor()**。



## 多线程定时任务如何配置？

下面介绍三种方案配置多线程下的定时任务。

### **1、重写SchedulingConfigurer#configureTasks()**

直接实现**SchedulingConfigurer**这个接口，设置**taskScheduler**，代码如下：

```java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        //设定一个长度10的定时任务线程池
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(10));
    }
}
```



### **2、通过配置开启**

Spring Boot quartz 已经提供了一个配置用来配置线程池的大小，如下；

```properties
spring.task.scheduling.pool.size=10
```

只需要在配置文件中添加如上的配置即可生效！



### **3、结合@Async**

**@Async**这个注解都用过，用来开启异步任务的，使用@Async这个注解之前一定是要先配置线程池的，配置如下：

```java
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor poolTaskExecutor = new ThreadPoolTaskExecutor();
        poolTaskExecutor.setCorePoolSize(4);
        poolTaskExecutor.setMaxPoolSize(6);
        // 设置线程活跃时间（秒）
        poolTaskExecutor.setKeepAliveSeconds(120);
        // 设置队列容量
        poolTaskExecutor.setQueueCapacity(40);
        poolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 等待所有任务结束后再关闭线程池
        poolTaskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        return poolTaskExecutor;
    }
```

然后在**@Scheduled**方法上标注**@Async**这个注解即可实现多线程定时任务，代码如下：

```java
	@Async
    @Scheduled(cron = "0/2 * * * * ? ")
    public void test2() {
        System.out.println("..................执行test2.................");
    }
```



## 总结

本篇文章介绍了Spring Boot 中 实现多线程定时任务的三种方案，你喜欢哪一种？

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！



