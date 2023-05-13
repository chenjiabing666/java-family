**大家好，我是不才陈某~**

任务调度是java项目中常用的一种组件，可以指定任务在何时进行触发，最熟悉的是spring框架里面的**quartz**；较流行的有一些分布式调度组件，比如**elastic-job**/**azkaban**,都是基于**quartz**二次开发的；

往期有篇文章介绍了分布式调度框架的核心逻辑：[聊聊分布式任务调度系统](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247509138&idx=1&sn=213b1855fb1832587d39844cbe42a49f&chksm=fcf77b5fcb80f249c1b5281b27a7b6908397d2c7666db494898a248f430ba1e13761a384e744&token=651972823&lang=zh_CN#rd)

今天介绍一款分布式的任务调度框架：**xxl-job**。

## 项目介绍

xxl-job是一款极容易学习上手的轻量级开源分布式调度框架,分为管理端和执行端两块,管理端负责配置任务信息以及查看任务执行日志,执行端只需要配置与管理端的连接信息就可以进行具体的任务逻辑开发了,目前版本还在持续迭代中,使用简单,功能强大,具体功能特性可以看下官方介绍。废话不多说,直接进入实战吧。

## 实战

### 1.服务端部署

从`https://github.com/xuxueli/xxl-job`下载项目,用mysql客户端工具Navicat执行项目根目录下doc/db/table_xxl_job.sql文件,库名自己可以自行修改,一共8张表,如下:

![](https://img.java-family.cn/20220529150032.png)

创建一个新的spring boot项目,将下载的xxl-job-admin目录下的文件以及pom.xml文件都拷贝到新建的项目中(如果不想新建项目可以直接用下载下来的项目进行修改部署),修改application.properties中的数据库连接信息。

![](https://img.java-family.cn/20220529150037.png)

小编是自己创建的新项目,需要手动改了pom.xml依赖xxl-job-core的版本为2.2.0

![](https://img.java-family.cn/20220529150047.png)

修改logback.xml中的日志输出路径。

![](https://img.java-family.cn/20220529150052.png)

好了,以上3步曲就搞定整个服务端配置了,启动项目,并访问`http://localhost:8080/xxl-job-admin/` ,默认管理员账号admin/123456进行登录。

![](https://img.java-family.cn/20220529150100.png)

这交互,可以啊,是不是很带感。

### 2.执行端配置

创建一个新的module,跟服务端一样,也需要修改下logback.xml以及在pom.xml添加xxl-job-core的依赖。

为了模拟分布式效果,小编创建了2个配置文件来区分2个执行服务。

**application-9998.properties**

![](https://img.java-family.cn/20220529145705.png)

**application-9999.properties**

![](https://img.java-family.cn/20220529145731.png)

细心的童鞋会发现只有server.port和xxl.job.executor.port不同,执行器服务跟spring boot一样,自带内嵌tomcat,也会暴露一个端口注册到服务端,进行高可用负载。

创建一个java config类,定义一个使用配置的XxlJobSpringExecutor执行类,如下

![](https://img.java-family.cn/20220529145846.png)![](https://img.java-family.cn/20220529145852.png)

配置2个启动配置,分别启动,效果如下:

![](https://img.java-family.cn/20220529150108.png)

![](https://img.java-family.cn/20220529150112.png)

完美启动2个服务,看下服务端平台是不是有这两台执行服务的注册信息。

![](https://img.java-family.cn/20220529150116.png)

注意:为了演示,事先创建了一个执行器,AppName一定要与配置文件中xxl.job.executor.appname一致。

### 3.任务开发

#### **3.1 基于方法注解任务**

话不多说,直接上代码把,毕竟代码是程序员最好的交流方式。

![](https://img.java-family.cn/20220529145928.png)

#### **3.2 基于api任务**

![](https://img.java-family.cn/20220529145952.png)

#### **3.3 分片广播任务**

![](https://img.java-family.cn/20220529150011.png)

上面是整理的比较实用的任务创建方式,个人偏好于注解形式,方法上加一个注解就完事了。

### 4.任务执行

剩下的就是傻白甜的界面操作了,走起。

#### **4.1 单任务执行**

创建一个路由策略为轮询的任务,指定corn表达式,并填入JobHandler为myJobAnnotationHandler,myJobAnnotationHandler其实就是spring IOC容器中管理bean的名称,有兴趣的童鞋可以看下源码。

![](https://img.java-family.cn/20220529150127.png)

为了演示效果,点击执行一次并进行任务参数输入。

![](https://img.java-family.cn/20220529150134.png)

![](https://img.java-family.cn/20220529150138.png)

轮询调用执行器服务效果如下:

![](https://img.java-family.cn/20220529150142.png)

![](https://img.java-family.cn/20220529150147.png)

#### **4.2 子任务执行**

更新任务,并指定子任务id为5,多个子任务的需要以逗号隔开

![](https://img.java-family.cn/20220529150151.png)



执行任务结果如下

![](https://img.java-family.cn/20220529150157.png)

#### **4.3 分片广播任务执行**

分片任务其实就是广播功能,每次触发,每个执行服务的业务执行类都会被调用,类似于kafka里面的不同消费组都要对同一个topic进行消费一样。

![](https://img.java-family.cn/20220529150201.png)

执行后的效果如下

![](https://img.java-family.cn/20220529150206.png)

![](https://img.java-family.cn/20220529150210.png)

太强势了,需要定时刷新项目中的配置信息,用这个方式很完美。

### 5.任务日志

任务日志其实是很重要的一块,方便回溯任务历史执行情况, 以便跟踪问题并矫正丢失的业务数据

![](https://img.java-family.cn/20220529150214.png)

查看调度备注,父子任务调度信息非常详细,子任务可以通过执行备注查看执行情况

![](https://img.java-family.cn/20220529150218.png)

![](https://img.java-family.cn/20220529150222.png)

查看控制台输出,里面的日志是执行器中XxlJobLogger类打印出来的

![](https://img.java-family.cn/20220529150227.png)



> 案例源码已经上传Github，关注公众号：**码猿技术专栏**，回复关键词：**9629** 获取！



## 通信底层介绍

xxl-job 使用 netty http 的方式进行通信，虽然也支持 Mina，jetty，netty tcp 等方式，但是代码里面固定写死的是 netty http。

## 通信整体流程

我以调度器通知执行器执行任务为例，绘制的活动图：

![活动图](https://img.java-family.cn/20220529150751.png)





## 惊艳的设计

看完了整个处理流程代码，设计上可以说独具匠心，将 netty，多线程的知识运用得行云流水。

我现在就将这些设计上出彩的点总结如下：

### **1. 使用动态代理模式，隐藏通信细节**

xxl-job 定义了两个接口 ExecutorBiz，AdminBiz，ExecutorBiz 接口中封装了向心跳，暂停，触发执行等操作，AdminBiz 封装了回调，注册，取消注册操作，接口的实现类中，并没有通信相关的处理。

XxlRpcReferenceBean 类的 getObject() 方法会生成一个代理类，这个代理类会进行远程通信。



### **2. 全异步处理**

执行器收到消息进行反序列化，并没有同步执行任务代码，而是将任务信息存储在 LinkedBlockingQueue 中，异步线程从这个队列中获取任务信息，然后执行。

而任务的处理结果，也不是说处理完之后，同步返回的，也是放到回调线程的阻塞队列中，异步的将处理结果返回回去。

这样处理的好处就是减少了 netty 工作线程的处理时间，提升了吞吐量。



### **3. 对异步处理的包装**

对异步处理进行了包装，代码看起来是同步调用的。

我们看下调度器，XxlJobTrigger 类触发任务执行的代码：

```java
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
    ReturnT<String> runResult = null;
    try {
        ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
        //这里面做了很多异步处理，最终同步得到处理结果
        runResult = executorBiz.run(triggerParam);
    } catch (Exception e) {
        logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
        runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
    }

    StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
    runResultSB.append("<br>address：").append(address);
    runResultSB.append("<br>code：").append(runResult.getCode());
    runResultSB.append("<br>msg：").append(runResult.getMsg());

    runResult.setMsg(runResultSB.toString());
    return runResult;
}
```

ExecutorBiz.run 方法我们说过了，是走的动态代理，和执行器进行通信，执行器执行结果也是异步处理完，才返回的，而这里看到的 run 方法是同步等待处理结果返回。

我们看下xxl-job是如何同步获取处理结果的：调度器向执行器发出消息后，该线程阻塞。等到执行器处理完毕后，将处理结果返回，唤醒被阻塞的线程，调用处拿到返回值。

动态代理代码如下：

```java
//代理类中的触发调用
if (CallType.SYNC == callType) {
   // future-response set
   XxlRpcFutureResponse futureResponse = new XxlRpcFutureResponse(invokerFactory, xxlRpcRequest, null);
   try {
      // do invoke
      client.asyncSend(finalAddress, xxlRpcRequest);

      // future get
      XxlRpcResponse xxlRpcResponse = futureResponse.get(timeout, TimeUnit.MILLISECONDS);
      if (xxlRpcResponse.getErrorMsg() != null) {
         throw new XxlRpcException(xxlRpcResponse.getErrorMsg());
      }
      return xxlRpcResponse.getResult();
   } catch (Exception e) {
      logger.info(">>>>>>>>>>> xxl-rpc, invoke error, address:{}, XxlRpcRequest{}", finalAddress, xxlRpcRequest);

      throw (e instanceof XxlRpcException)?e:new XxlRpcException(e);
   } finally{
      // future-response remove
      futureResponse.removeInvokerFuture();
   }
} 
```



XxlRpcFutureResponse 类中实现了线程的等待，和线程唤醒的处理：

```java
//返回结果，唤醒线程
public void setResponse(XxlRpcResponse response) {
   this.response = response;
   synchronized (lock) {
      done = true;
      lock.notifyAll();
   }
}

@Override
    public XxlRpcResponse get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        if (!done) {
            synchronized (lock) {
                try {
                    if (timeout < 0) {
            //线程阻塞
                        lock.wait();
                    } else {
                        long timeoutMillis = (TimeUnit.MILLISECONDS==unit)?timeout:TimeUnit.MILLISECONDS.convert(timeout , unit);
                        lock.wait(timeoutMillis);
                    }
                } catch (InterruptedException e) {
                    throw e;
                }
            }
        }

        if (!done) {
            throw new XxlRpcException("xxl-rpc, request timeout at:"+ System.currentTimeMillis() +", request:" + request.toString());
        }
        return response;
    }
```



有的同学可能会问了，调度器接收到返回结果，怎么确定唤醒哪个线程呢？

每一次远程调用，都会生成 uuid 的请求 id，这个 id 是在整个调用过程中一直传递的，就像一把钥匙，在你回家的的时候，拿着它就带开门。

这里拿着请求 id 这把钥匙，就能找到对应的 XxlRpcFutureResponse，然后调用 setResponse 方法，设置返回值，唤醒线程。

```java
public void notifyInvokerFuture(String requestId, final XxlRpcResponse xxlRpcResponse){

    // 通过requestId找到XxlRpcFutureResponse，
    final XxlRpcFutureResponse futureResponse = futureResponsePool.get(requestId);
    if (futureResponse == null) {
        return;
    }
    if (futureResponse.getInvokeCallback()!=null) {

        // callback type
        try {
            executeResponseCallback(new Runnable() {
                @Override
                public void run() {
                    if (xxlRpcResponse.getErrorMsg() != null) {
                        futureResponse.getInvokeCallback().onFailure(new XxlRpcException(xxlRpcResponse.getErrorMsg()));
                    } else {
                        futureResponse.getInvokeCallback().onSuccess(xxlRpcResponse.getResult());
                    }
                }
            });
        }catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
    } else {
        // 里面调用lock的notify方法
        futureResponse.setResponse(xxlRpcResponse);
    }

    // do remove
    futureResponsePool.remove(requestId);

}
```



> 案例源码已经上传Github，关注公众号：**码猿技术专栏**，回复关键词：**9629** 获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！
