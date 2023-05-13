# Spring Cloud Gateway 聚合 Swagger 文章
今天这篇文章介绍一下微服务如何聚合**Swagger**实现接口文档管理。

文章目录如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/15.png)



## 为什么需要聚合？

微服务模块众多，如果不聚合文档，则访问每个服务的API文档都需要**单独**访问一个Swagger UI界面，这么做客户端能否接受？

**反正作为强迫症的我是接受不了.......**

既然使用了微服务，就应该有**统一的API文档入口**。



## 如何聚合？

统一的文档入口显然应该聚合到网关中，通过**网关**的入口统一映射到各个模块。

![演示](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/1.gif)



本文采用**Spring Cloud Gateway** 聚合 **Swagger** 的 方式 生成API文档。

案例源码结构如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/2.png)



本文只介绍如何聚合Swagger，关于网关、注册中心等内容不再介绍，有不了解的看陈某前面文章。



## 单个服务如何聚合Swagger？

这里的单个服务不包括网关，网关需要单独配置。

单个服务聚合其实很简单，就是普通的Spring Boot 整合 Swagger，但是微服务模块众多，不能每个微服都整合一番，因此可以自定义一个**swagger-starter**，之后每个微服务都依赖这个starter即可。

详细的步骤如下：



### 1、创建swagger-starter

自定义starter这里就不再介绍了，都是基础的知识；

目录结构如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/3.png)

**1、添加依赖**

对于Swagger原生的UI界面陈某不太喜欢，因此使用了一款看起来还不错的UI界面，依赖如下：

```xml
<!--swagger-->
<dependency>
   <groupId>io.springfox</groupId>
   <artifactId>springfox-boot-starter</artifactId>
</dependency>

<!--swagger-ui  这里是用了一个好看一点ui界面-->
<dependency>
   <groupId>com.github.xiaoymin</groupId>
   <artifactId>swagger-bootstrap-ui</artifactId>
</dependency>
```



> 对于UI界面，每个人审美不同，选择自己喜欢的就好。



**2、自动配置类配置Swagger**

陈某是将每个服务的API信息抽离出一个属性类**SwaggerProperties**，后续只需要在每个服务的配置文件中指定即可。

```java
@Data
@ConfigurationProperties(prefix = SwaggerProperties.PREFIX)
@Component
@EnableConfigurationProperties
public class SwaggerProperties {
    public static final String PREFIX="spring.swagger";

    //包
    private String basePackage;

    //作者相关信息
    private Author author;

    //API的相关信息
    private ApiInfo apiInfo;

    @Data
    public static class ApiInfo{
        String title;
        String description;
        String version;
        String termsOfServiceUrl;
        String license;
        String licenseUrl;
    }
    @Data
    public static class Author{
        private String name;

        private String email;

        private String url;
    }
}
```



对于Swagger的配置其实很简单，分为如下部分：

1. API文档基本信息配置
2. 授权信息配置（基于OAuth2的认证配置）



API文档配置无非就是配置文档的基本信息，比如文档标题、作者、联系方式.....

代码如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/4.png)



授权信息配置也很简单，就是在全局信息的请求头中配置一个能够放置令牌的地方，代码如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/5.png)

此处对应UI界面的地方如下图：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/6.png)

只需要将获取token令牌设置到这里即可。

好了，swagger-starter关键代码就介绍完了，详细配置见源码。

> 案例源码已上传GitHub，关注公众号：**码猿技术专栏**，回复关键：**9528** 获取！



### 2、微服务引用swagger-starter

单个微服务引用就很简单了，只需要添加如下依赖：

```xml
<dependency>
  <groupId>cn.myjszl</groupId>
  <artifactId>swagger-starter</artifactId>
</dependency>
```

接下来只需要在配置文件配置**API**相关的信息即可，比如订单服务的配置如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/7.png)

好了，至此单个服务的配置完成了。

此时我们可以验证一下，直接访问：http://localhost:3002/swagger-order-boot/v2/api-docs，结果如下图：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/8.png)



## 网关如何聚合Swagger？

网关聚合的思想很简单，就是从路由中获取微服务的访问地址，然后拼接上 **/v2/api-docs** 即可。

同样的还是要添加Swagger的两个依赖，如下：

```xml
<!--swagger-->
<dependency>
   <groupId>io.springfox</groupId>
   <artifactId>springfox-boot-starter</artifactId>
</dependency>

<!--swagger-ui  这里是用了一个好看一点ui界面-->
<dependency>
   <groupId>com.github.xiaoymin</groupId>
   <artifactId>swagger-bootstrap-ui</artifactId>
</dependency>
```



创建**GatewaySwaggerResourcesProvider**实现**SwaggerResourcesProvider**，重写其中的get方法，代码如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/9.png)



> 案例源码已上传GitHub，关注公众号：**码猿技术专栏**，回复关键：**9528** 获取！

好了，网关的配置这里就完成了。

此时启动网关、订单、库存服务，直接访问网关的文档：http://localhost:3001/doc.html，结果如下图：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/10.png)



## API文档好用的功能介绍

不得不说这款Swagger UI 界面还是比较简单易用的，个人用起来还不错。

### 1、搜索功能

在右上角的搜索功能可以根据接口描述搜索相关的接口信息，如下图：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/11.png)



### 2、离线文档

可以直接拷贝文档的MarkDown形式转换成Html或者PDF生成离线文档，如下图：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/12.png)



### 3、令牌配置

在访问需要认证的接口时，可以通过配置令牌，这样令牌将会全局生效，不必每个请求都要配置一遍，如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/13.png)



### 4、配置缓存

该文档的所有配置，包括请求参数、授权令牌等信息都是缓存的，也就是说配置一次，下次再打开的时候也是默认存在的。



### 5、全局参数配置

对于一些全局的参数，比如请求头中需要携带请求客户端、版本号等信息，可以在全局参数中配置，如下：

![](https://img.java-family.cn/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%81%9A%E5%90%88API%E6%96%87%E6%A1%A3/14.png)



## 总结

本篇文章介绍了微服务集成网关聚合Swagger文档，开发中非常实用。

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

如果这篇文章对你有所帮助，或者有所启发的话，帮忙**点赞**、**在看**、**转发**、**收藏**，你的支持就是我坚持下去的最大动力！

关注公众号：**【码猿技术专栏】**，公众号内有超赞的粉丝福利，回复：**加群**，可以加入技术讨论群，和大家一起讨论技术，吹牛逼！
