## 前言

最近频繁被`Swagger 3.0`刷屏，官方表示这是一个突破性的变更，有很多的亮点，我还真不太相信，今天来带大家尝尝鲜，看看这碗汤到底鲜不鲜....

## 官方文档如何说？

该项目开源在`Github`上，地址：[https://github.com/springfox/springfox](https://github.com/springfox/springfox)。

`Swagger 3.0`有何改动？官方文档总结如下几点：
1. 删除了对`springfox-swagger2`的依赖
2. 删除所有`@EnableSwagger2...`注解
3. 添加了`springfox-boot-starter`依赖项
4. 移除了`guava`等第三方依赖
5. 文档访问地址改变了，改成了`http://ip:port/project/swagger-ui/index.html`。

> 姑且看到这里，各位初始感觉如何？

![](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/1.png)

既然人家更新出来了，咱不能不捧场，下面就介绍下`Spring Boot`如何整合`Swagger 3.0`吧。


## Spring Boot版本说明
作者使用`Spring Boot`的版本是`2.3.5.RELEASE`

## 添加依赖
`Swagger 3.0`已经有了与Spring Boot整合的启动器，只需要添加以下依赖：

```xml
  <dependency>
       <groupId>io.springfox</groupId>
       <artifactId>springfox-boot-starter</artifactId>
       <version>3.0.0</version>
  </dependency>
```

## springfox-boot-starter做了什么？
`Swagger 3.0`主推的一大特色就是这个启动器，那么这个启动器做了什么呢？

> **记住**：启动器的一切逻辑都在自动配置类中。

找到`springfox-boot-starter`的自动配置类，在`/META-INF/spring.factories`文件中，如下：

![](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/2.png)

从上图可以知道，自动配置类就是`OpenApiAutoConfiguration`，源码如下：
```java
@Configuration
@EnableConfigurationProperties(SpringfoxConfigurationProperties.class)
@ConditionalOnProperty(value = "springfox.documentation.enabled", havingValue = "true", matchIfMissing = true)
@Import({
    OpenApiDocumentationConfiguration.class,
    SpringDataRestConfiguration.class,
    BeanValidatorPluginsConfiguration.class,
    Swagger2DocumentationConfiguration.class,
    SwaggerUiWebFluxConfiguration.class,
    SwaggerUiWebMvcConfiguration.class
})
@AutoConfigureAfter({ WebMvcAutoConfiguration.class, JacksonAutoConfiguration.class,
    HttpMessageConvertersAutoConfiguration.class, RepositoryRestMvcAutoConfiguration.class })
public class OpenApiAutoConfiguration {

}
```

敢情这个自动配置类啥也没干，就光导入了几个配置类(`@Import`)以及开启了属性配置(`@EnableConfigurationProperties`)。

![3](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/3.jpg)

> **重点**：记住`OpenApiDocumentationConfiguration`这个配置类，初步看来这是个BUG，本人也不想深入，里面的代码写的实在拙劣，注释都不写。


## 撸起袖子就是干？
说真的，还是和以前一样，真的没什么太大的改变，按照文档的步骤一步步来。

### 定制一个基本的文档示例
一切的东西还是需要配置类手动配置，说真的，我以为会在全局配置文件中自己配置就行了。哎，想多了。配置类如下：
```java
@EnableOpenApi
@Configuration
@EnableConfigurationProperties(value = {SwaggerProperties.class})
public class SwaggerConfig {
  /**
     * 配置属性
     */
    @Autowired
    private SwaggerProperties properties;

    @Bean
    public Docket frontApi() {
        return new Docket(DocumentationType.OAS_30)
                //是否开启，根据环境配置
                .enable(properties.getFront().getEnable())
                .groupName(properties.getFront().getGroupName())
                .apiInfo(frontApiInfo())
                .select()
                //指定扫描的包
                .apis(RequestHandlerSelectors.basePackage(properties.getFront().getBasePackage()))
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 前台API信息
     */
    private ApiInfo frontApiInfo() {
        return new ApiInfoBuilder()
                .title(properties.getFront().getTitle())
                .description(properties.getFront().getDescription())
                .version(properties.getFront().getVersion())
                .contact(    //添加开发者的一些信息
                        new Contact(properties.getFront().getContactName(), properties.getFront().getContactUrl(),
                                properties.getFront().getContactEmail()))
                .build();
    }
}
```

`@EnableOpenApi`这个注解文档解释如下：
```doc
Indicates that Swagger support should be enabled.
This should be applied to a Spring java config and should have an accompanying '@Configuration' annotation.
Loads all required beans defined in @see SpringSwaggerConfig
```

什么意思呢？大致意思就是**只有在配置类标注了`@EnableOpenApi`这个注解才会生成Swagger文档**。

`@EnableConfigurationProperties`这个注解使开启自定义的属性配置，这是作者自定义的`Swagger`配置。

> 总之还是和之前一样配置，根据官方文档要求，需要在配置类上加一个`@EnableOpenApi`注解。

### 文档如何分组？
我们都知道，一个项目可能分为`前台`，`后台`，`APP端`，`小程序端`.....每个端的接口可能还相同，不可能全部放在一起吧，肯定是要区分开的。

因此，实际开发中文档肯定是要分组的。

分组其实很简单，`Swagger`向`IOC`中注入一个`Docket`即为一个组的文档，其中有个`groupName()`方法指定分组的名称。

因此只需要注入多个`Docket`指定不同的组名即可，当然，这些文档的标题、描述、扫描的路径都是可以不同定制的。

如下配置两个`Docket`，分为前台和后台，配置类如下：
```java
@EnableOpenApi
@Configuration
@EnableConfigurationProperties(value = {SwaggerProperties.class})
public class SwaggerConfig {
  /**
     * 配置属性
     */
    @Autowired
    private SwaggerProperties properties;

    @Bean
    public Docket frontApi() {
        return new Docket(DocumentationType.OAS_30)
                //是否开启，根据环境配置
                .enable(properties.getFront().getEnable())
                .groupName(properties.getFront().getGroupName())
                .apiInfo(frontApiInfo())
                .select()
                //指定扫描的包
                .apis(RequestHandlerSelectors.basePackage(properties.getFront().getBasePackage()))
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 前台API信息
     */
    private ApiInfo frontApiInfo() {
        return new ApiInfoBuilder()
                .title(properties.getFront().getTitle())
                .description(properties.getFront().getDescription())
                .version(properties.getFront().getVersion())
                .contact(    //添加开发者的一些信息
                        new Contact(properties.getFront().getContactName(), properties.getFront().getContactUrl(),
                                properties.getFront().getContactEmail()))
                .build();
    }
    
    /**
     * 后台API
     */
    @Bean
    public Docket backApi() {
        return new Docket(DocumentationType.OAS_30)
                //是否开启，根据环境配置
                .enable(properties.getBack().getEnable())
                .groupName("后台管理")
                .apiInfo(backApiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage(properties.getBack().getBasePackage()))
                .paths(PathSelectors.any())
                .build();
    }
    
    /**
     * 后台API信息
     */
    private ApiInfo backApiInfo() {
        return new ApiInfoBuilder()
                .title(properties.getBack().getTitle())
                .description(properties.getBack().getDescription())
                .version(properties.getBack().getVersion())
                .contact(    //添加开发者的一些信息
                        new Contact(properties.getBack().getContactName(), properties.getBack().getContactUrl(),
                                properties.getBack().getContactEmail()))
                .build();
    }
    
}
```

属性配置文件`SwaggerProperties`如下，分为前台和后台两个不同属性的配置：
```java
/**
 * swagger的属性配置类
 */
@ConfigurationProperties(prefix = "spring.swagger")
@Data
public class SwaggerProperties {

    /**
     * 前台接口配置
     */
    private SwaggerEntity front;

    /**
     * 后台接口配置
     */
    private SwaggerEntity back;

    @Data
    public static class SwaggerEntity {
        private String groupName;
        private String basePackage;
        private String title;
        private String description;
        private String contactName;
        private String contactEmail;
        private String contactUrl;
        private String version;
        private Boolean enable;
    }
}
```

此时的文档截图如下，可以看到有了两个不同的分组：

![](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/4.png)


### 如何添加授权信息？

现在项目API肯定都需要权限认证，否则不能访问，比如请求携带一个`TOKEN`。

在Swagger中也是可以配置认证信息，这样在每次请求将会默认携带上。

在`Docket`中有如下两个方法指定授权信息，分别是`securitySchemes()`和`securityContexts()`。在配置类中的配置如下，在构建Docket的时候设置进去即可：
```java

    @Bean
    public Docket frontApi() {
        RequestParameter parameter = new RequestParameterBuilder()
                .name("platform")
                .description("请求头")
                .in(ParameterType.HEADER)
                .required(true)
                .build();
        List<RequestParameter> parameters = Collections.singletonList(parameter);
        return new Docket(DocumentationType.OAS_30)
                //是否开启，根据环境配置
                .enable(properties.getFront().getEnable())
                .groupName(properties.getFront().getGroupName())
                .apiInfo(frontApiInfo())
                .select()
                //指定扫描的包
                .apis(RequestHandlerSelectors.basePackage(properties.getFront().getBasePackage()))
                .paths(PathSelectors.any())
                .build()
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts());
    }

    /**
     * 设置授权信息
     */
    private List<SecurityScheme> securitySchemes() {
        ApiKey apiKey = new ApiKey("BASE_TOKEN", "token", In.HEADER.toValue());
        return Collections.singletonList(apiKey);
    }

    /**
     * 授权信息全局应用
     */
    private List<SecurityContext> securityContexts() {
        return Collections.singletonList(
                SecurityContext.builder()
                        .securityReferences(Collections.singletonList(new SecurityReference("BASE_TOKEN", new AuthorizationScope[]{new AuthorizationScope("global", "")})))
                        .build()
        );
    }
```

以上配置成功后，在Swagger文档的页面中将会有`Authorize`按钮，只需要将请求头添加进去即可。如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/5.png)

### 如何携带公共的请求参数？
不同的架构可能发请求的时候除了携带`TOKEN`，还会携带不同的参数，比如请求的平台，版本等等，这些每个请求都要携带的参数称之为公共参数。

那么如何在`Swagger`中定义公共的参数呢？比如在请求头中携带。

在`Docket`中的方法`globalRequestParameters()`可以设置公共的请求参数，接收的参数是一个`List<RequestParameter>`，因此只需要构建一个`RequestParameter`集合即可，如下：
```java
@Bean
public Docket frontApi() {
   //构建一个公共请求参数platform，放在在header
   RequestParameter parameter = new RequestParameterBuilder()
      //参数名称
      .name("platform")
      //描述
      .description("请求的平台")
      //放在header中
      .in(ParameterType.HEADER)
      //是否必传
      .required(true)
      .build();
      //构建一个请求参数集合
      List<RequestParameter> parameters = Collections.singletonList(parameter);
        return new Docket(DocumentationType.OAS_30)
                .....
                .build()
                .globalRequestParameters(parameters);
    }
```

以上配置完成，将会在每个接口中看到一个请求头，如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E7%AC%AC%E5%8D%81%E5%85%AD%E5%BC%B9%EF%BC%8C%E6%95%B4%E5%90%88Swagger3.0/6.png)

## 粗略是一个BUG
作者在介绍自动配置类的时候提到了一嘴，现在来简单分析下。

`OpenApiAutoConfiguration`这个自动配置类中已经导入`OpenApiDocumentationConfiguration`这个配置类，如下一段代码：
```java
@Import({
    OpenApiDocumentationConfiguration.class,
    ......
})
```

`@EnableOpenApi`的源码如下：
```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = {java.lang.annotation.ElementType.TYPE})
@Documented
@Import(OpenApiDocumentationConfiguration.class)
public @interface EnableOpenApi {
}
```

从源码可以看出：`@EnableOpenApi`这个注解的作用就是导入`OpenApiDocumentationConfiguration`这个配置类，纳尼？？？

既然已经在自动配置类`OpenApiAutoConfiguration`导入了，那么无论需不需要在配置类上标注`@EnableOpenApi`注解不都会开启`Swagger`支持吗？

**测试一下**：不在配置类上标注`@EnableOpenApi`这个注解，看看是否`Swagger`运行正常。结果在意料之中，还是能够正常运行。

> **总结**：作者只是大致分析了下，这可能是个`BUG`亦或是后续有其他的目的，至于结果如此，不想验证了，没什么意思。


## 总结

这篇文章也是尝了个鲜，个人感觉不太香，有点失望。你喜欢吗？

> `Spring Boot` 整合的源码已经上传，需要的朋友回复关键词**Swagger3.0**获取。
