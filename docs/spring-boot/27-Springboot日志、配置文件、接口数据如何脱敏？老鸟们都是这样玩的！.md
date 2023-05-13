## 一、前言

核心隐私数据无论对于企业还是用户来说尤其重要，因此要想办法杜绝各种隐私数据的泄漏。下面陈某带大家从以下三个方面讲解一下隐私数据如何脱敏，也是日常开发中需要注意的：

1. 配置文件数据脱敏
2. 接口返回数据脱敏
3. 日志文件数据脱敏

文章目录如下：

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/6.png)

## 二、配置文件如何脱敏？

经常会遇到这样一种情况：项目的配置文件中总有一些敏感信息，比如数据源的url、用户名、密码....这些信息一旦被暴露那么整个数据库都将会被泄漏，那么如何将这些配置隐藏呢？

以前都是手动将加密之后的配置写入到配置文件中，提取的时候再手动解密，当然这是一种思路，也能解决问题，但是每次都要手动加密、解密不觉得麻烦吗？

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/7.jpg)

今天介绍一种方案，让你在无感知的情况下实现配置文件的加密、解密。利用一款开源插件：`jasypt-spring-boot`。项目地址如下：

```java
https://github.com/ulisesbocchio/jasypt-spring-boot
```

使用方法很简单，整合Spring Boot 只需要添加一个`starter`。

### 1. 添加依赖

```xml
<dependency>
        <groupId>com.github.ulisesbocchio</groupId>
        <artifactId>jasypt-spring-boot-starter</artifactId>
        <version>3.0.3</version>
</dependency>		
```

### 2. 配置秘钥

在配置文件中添加一个加密的秘钥（任意），如下：

```yaml
jasypt:
  encryptor:
    password: Y6M9fAJQdU7jNp5MW
```

当然将秘钥直接放在配置文件中也是不安全的，我们可以在项目启动的时候配置秘钥，命令如下：

```shell
java -jar xxx.jar  -Djasypt.encryptor.password=Y6M9fAJQdU7jNp5MW
```

### 3. 生成加密后的数据

这一步骤是将配置明文进行加密，代码如下：

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class SpringbootJasyptApplicationTests {

    /**
     * 注入加密方法
     */
    @Autowired
    private StringEncryptor encryptor;

    /**
     * 手动生成密文，此处演示了url，user，password
     */
    @Test
    public void encrypt() {
        String url = encryptor.encrypt("jdbc\\:mysql\\://127.0.0.1\\:3306/test?useUnicode\\=true&characterEncoding\\=UTF-8&zeroDateTimeBehavior\\=convertToNull&useSSL\\=false&allowMultiQueries\\=true&serverTimezone=Asia/Shanghai");
        String name = encryptor.encrypt("root");
        String password = encryptor.encrypt("123456");
        System.out.println("database url: " + url);
        System.out.println("database name: " + name);
        System.out.println("database password: " + password);
        Assert.assertTrue(url.length() > 0);
        Assert.assertTrue(name.length() > 0);
        Assert.assertTrue(password.length() > 0);
    }
}
```

上述代码对数据源的url、user、password进行了明文加密，输出的结果如下：

```properties
database url: szkFDG56WcAOzG2utv0m2aoAvNFH5g3DXz0o6joZjT26Y5WNA+1Z+pQFpyhFBokqOp2jsFtB+P9b3gB601rfas3dSfvS8Bgo3MyP1nojJgVp6gCVi+B/XUs0keXPn+pbX/19HrlUN1LeEweHS/LCRZslhWJCsIXTwZo1PlpXRv3Vyhf2OEzzKLm3mIAYj51CrEaN3w5cMiCESlwvKUhpAJVz/uXQJ1spLUAMuXCKKrXM/6dSRnWyTtdFRost5cChEU9uRjw5M+8HU3BLemtcK0vM8iYDjEi5zDbZtwxD3hA=

database name: L8I2RqYPptEtQNL4x8VhRVakSUdlsTGzEND/3TOnVTYPWe0ZnWsW0/5JdUsw9ulm

database password: EJYCSbBL8Pmf2HubIH7dHhpfDZcLyJCEGMR9jAV3apJtvFtx9TVdhUPsAxjQ2pnJ
```

### 4. 将加密后的密文写入配置

`jasypt`默认使用`ENC()`包裹，此时的数据源配置如下：

```yaml
spring:
  datasource:
    #   数据源基本配置
    username: ENC(L8I2RqYPptEtQNL4x8VhRVakSUdlsTGzEND/3TOnVTYPWe0ZnWsW0/5JdUsw9ulm)
    password: ENC(EJYCSbBL8Pmf2HubIH7dHhpfDZcLyJCEGMR9jAV3apJtvFtx9TVdhUPsAxjQ2pnJ)
    driver-class-name: com.mysql.jdbc.Driver
    url: ENC(szkFDG56WcAOzG2utv0m2aoAvNFH5g3DXz0o6joZjT26Y5WNA+1Z+pQFpyhFBokqOp2jsFtB+P9b3gB601rfas3dSfvS8Bgo3MyP1nojJgVp6gCVi+B/XUs0keXPn+pbX/19HrlUN1LeEweHS/LCRZslhWJCsIXTwZo1PlpXRv3Vyhf2OEzzKLm3mIAYj51CrEaN3w5cMiCESlwvKUhpAJVz/uXQJ1spLUAMuXCKKrXM/6dSRnWyTtdFRost5cChEU9uRjw5M+8HU3BLemtcK0vM8iYDjEi5zDbZtwxD3hA=)
    type: com.alibaba.druid.pool.DruidDataSource
```

上述配置是使用默认的`prefix=ENC(`、`suffix=)`，当然我们可以根据自己的要求更改，只需要在配置文件中更改即可，如下：

```yaml
jasypt:
  encryptor:
    ## 指定前缀、后缀
    property:
      prefix: 'PASS('
      suffix: ')'
```

那么此时的配置就必须使用`PASS()`包裹才会被解密，如下：

```yaml
spring:
  datasource:
    #   数据源基本配置
    username: PASS(L8I2RqYPptEtQNL4x8VhRVakSUdlsTGzEND/3TOnVTYPWe0ZnWsW0/5JdUsw9ulm)
    password: PASS(EJYCSbBL8Pmf2HubIH7dHhpfDZcLyJCEGMR9jAV3apJtvFtx9TVdhUPsAxjQ2pnJ)
    driver-class-name: com.mysql.jdbc.Driver
    url: PASS(szkFDG56WcAOzG2utv0m2aoAvNFH5g3DXz0o6joZjT26Y5WNA+1Z+pQFpyhFBokqOp2jsFtB+P9b3gB601rfas3dSfvS8Bgo3MyP1nojJgVp6gCVi+B/XUs0keXPn+pbX/19HrlUN1LeEweHS/LCRZslhWJCsIXTwZo1PlpXRv3Vyhf2OEzzKLm3mIAYj51CrEaN3w5cMiCESlwvKUhpAJVz/uXQJ1spLUAMuXCKKrXM/6dSRnWyTtdFRost5cChEU9uRjw5M+8HU3BLemtcK0vM8iYDjEi5zDbZtwxD3hA=)
    type: com.alibaba.druid.pool.DruidDataSource
```

### 5. 总结

jasypt还有许多高级用法，比如可以自己配置加密算法，具体的操作可以参考Github上的文档。



## 三、接口返回数据如何脱敏？

通常接口返回值中的一些敏感数据也是要脱敏的，比如身份证号、手机号码、地址.....通常的手段就是用`*`隐藏一部分数据，当然也可以根据自己需求定制。

言归正传，如何优雅的实现呢？有两种实现方案，如下：

- 整合Mybatis插件，在查询的时候针对特定的字段进行脱敏

- 整合Jackson，在序列化阶段对特定字段进行脱敏

- 基于`Sharding Sphere`实现数据脱敏，查看之前的文章：[基于Sharding Sphere实现数据“一键脱敏”](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247493021&idx=2&sn=f27a4eeb6dd3eb5b1551eabf6e31739e&chksm=fcf73a50cb80b3465e6f7cf1fd841bfec098fcde47c587a2c35e7793bad3b33ebe8f2433d3ff&token=1439687168&lang=zh_CN#rd)

第一种方案网上很多实现方式，下面演示第二种，整合Jackson。

### 1. 自定义一个Jackson注解

需要自定义一个脱敏注解，一旦有属性被标注，则进行对应得脱敏，如下：

```java
/**
 * 自定义jackson注解，标注在属性上
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@JacksonAnnotationsInside
@JsonSerialize(using = SensitiveJsonSerializer.class)
public @interface Sensitive {
    //脱敏策略
    SensitiveStrategy strategy();
}
```

### 2. 定制脱敏策略

针对项目需求，定制不同字段的脱敏规则，比如手机号中间几位用`*`替代，如下：

```java
/**
 * 脱敏策略，枚举类，针对不同的数据定制特定的策略
 */
public enum SensitiveStrategy {
    /**
     * 用户名
     */
    USERNAME(s -> s.replaceAll("(\\S)\\S(\\S*)", "$1*$2")),
    /**
     * 身份证
     */
    ID_CARD(s -> s.replaceAll("(\\d{4})\\d{10}(\\w{4})", "$1****$2")),
    /**
     * 手机号
     */
    PHONE(s -> s.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2")),
    /**
     * 地址
     */
    ADDRESS(s -> s.replaceAll("(\\S{3})\\S{2}(\\S*)\\S{2}", "$1****$2****"));


    private final Function<String, String> desensitizer;

    SensitiveStrategy(Function<String, String> desensitizer) {
        this.desensitizer = desensitizer;
    }

    public Function<String, String> desensitizer() {
        return desensitizer;
    }
}
```

以上只是提供了部分，具体根据自己项目要求进行配置。

### 3. 定制JSON序列化实现

下面将是重要实现，对标注注解`@Sensitive`的字段进行脱敏，实现如下：

```java
/**
 * 序列化注解自定义实现
 * JsonSerializer<String>：指定String 类型，serialize()方法用于将修改后的数据载入
 */
public class SensitiveJsonSerializer extends JsonSerializer<String> implements ContextualSerializer {
    private SensitiveStrategy strategy;

    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(strategy.desensitizer().apply(value));
    }

    /**
     * 获取属性上的注解属性
     */
    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) throws JsonMappingException {

        Sensitive annotation = property.getAnnotation(Sensitive.class);
        if (Objects.nonNull(annotation)&&Objects.equals(String.class, property.getType().getRawClass())) {
            this.strategy = annotation.strategy();
            return this;
        }
        return prov.findValueSerializer(property.getType(), property);

    }
}
```

### 4. 定义Person类，对其数据脱敏

使用注解`@Sensitive`注解进行数据脱敏，代码如下：

```java
@Data
public class Person {
    /**
     * 真实姓名
     */
    @Sensitive(strategy = SensitiveStrategy.USERNAME)
    private String realName;
    /**
     * 地址
     */
    @Sensitive(strategy = SensitiveStrategy.ADDRESS)
    private String address;
    /**
     * 电话号码
     */
    @Sensitive(strategy = SensitiveStrategy.PHONE)
    private String phoneNumber;
    /**
     * 身份证号码
     */
    @Sensitive(strategy = SensitiveStrategy.ID_CARD)
    private String idCard;
}
```



### 5. 模拟接口测试

以上4个步骤完成了数据脱敏的Jackson注解，下面写个controller进行测试，代码如下：

```java
@RestController
public class TestController {
    @GetMapping("/test")
    public Person test(){
        Person user = new Person();
        user.setRealName("不才陈某");
        user.setPhoneNumber("19796328206");
        user.setAddress("浙江省杭州市温州市....");
        user.setIdCard("4333333333334334333");
        return user;
    }
}
```

调用接口查看数据有没有正常脱敏，结果如下：

```json
{
    "realName": "不*陈某",
    "address": "浙江省****市温州市..****",
    "phoneNumber": "197****8206",
    "idCard": "4333****34333"
}
```

### 6. 总结

数据脱敏有很多种实现方式，关键是哪种更加适合，哪种更加优雅.....



## 四、日志文件如何数据脱敏？

上面讲了配置文件、接口返回值的数据脱敏，现在总该轮到日志脱敏了。项目中总避免不了打印日志，肯定会涉及到一些敏感数据被明文打印出来，那么此时就需要过滤掉这些敏感数据（身份证、号码、用户名.....）。

关于Spring Boot 日志方面的问题有不理解的可以看我之前的文章：[Spring Boot第三弹，一文带你搞懂日志如何配置？](https://mp.weixin.qq.com/s/5hyGi6X6BhfCC5kTEvUyFg)、[Spring Boot第二弹，配置文件怎么造？](https://mp.weixin.qq.com/s/ute0JxWgqPe4VvKMbUQMNw)。

下面以**log4j2**这款日志为例讲解一下日志如何脱敏，其他日志框架大致思路一样。

### 1. 添加log4j2日志依赖

Spring Boot 默认日志框架是logback，但是我们可以切换到log4j2，依赖如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 去掉springboot默认配置 -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!--使用log4j2替换 LogBack-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

### 2. 在/resource目录下新建log4j2.xml配置

log4j2的日志配置很简单，只需要在`/resource`文件夹下新建一个`log4j2.xml`配置文件，内容如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/1.png)

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/2.png)

关于每个节点如何配置，含义是什么，在我上面的两篇文章中有详细的介绍。

> 上图的配置并没有实现数据脱敏，这是普通的配置，使用的是**PatternLayout**

### 3.  自定义PatternLayout实现数据脱敏

**步骤**2中的配置使用的是`PatternLayout`实现日志的格式，那么我们也可以自定义一个PatternLayout来实现日志的过滤脱敏。

PatternLayout的类图继承关系如下：

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/3.png)



从上图中可以清楚的看出来，PatternLayout继承了一个抽象类`AbstractStringLayout`，因此想要自定义只需要继承这个抽象类即可。

**1、创建CustomPatternLayout，继承抽象类AbstractStringLayout**

代码如下：

```java
/**
 * log4j2 脱敏插件
 * 继承AbstractStringLayout
 **/
@Plugin(name = "CustomPatternLayout", category = Node.CATEGORY, elementType = Layout.ELEMENT_TYPE, printObject = true)
public class CustomPatternLayout extends AbstractStringLayout {


    public final static Logger logger = LoggerFactory.getLogger(CustomPatternLayout.class);
    private PatternLayout patternLayout;


    protected CustomPatternLayout(Charset charset, String pattern) {
        super(charset);
        patternLayout = PatternLayout.newBuilder().withPattern(pattern).build();
        initRule();
    }

    /**
     * 要匹配的正则表达式map
     */
    private static Map<String, Pattern> REG_PATTERN_MAP = new HashMap<>();
    private static Map<String, String> KEY_REG_MAP = new HashMap<>();


    private void initRule() {
        try {
            if (MapUtils.isEmpty(Log4j2Rule.regularMap)) {
                return;
            }
            Log4j2Rule.regularMap.forEach((a, b) -> {
                if (StringUtils.isNotBlank(a)) {
                    Map<String, String> collect = Arrays.stream(a.split(",")).collect(Collectors.toMap(c -> c, w -> b, (key1, key2) -> key1));
                    KEY_REG_MAP.putAll(collect);
                }
                Pattern compile = Pattern.compile(b);
                REG_PATTERN_MAP.put(b, compile);
            });

        } catch (Exception e) {
            logger.info(">>>>>> 初始化日志脱敏规则失败 ERROR：{}", e);
        }

    }

    /**
     * 处理日志信息，进行脱敏
     * 1.判断配置文件中是否已经配置需要脱敏字段
     * 2.判断内容是否有需要脱敏的敏感信息
     * 2.1 没有需要脱敏信息直接返回
     * 2.2 处理: 身份证 ,姓名,手机号敏感信息
     */
    public String hideMarkLog(String logStr) {
        try {
            //1.判断配置文件中是否已经配置需要脱敏字段
            if (StringUtils.isBlank(logStr) || MapUtils.isEmpty(KEY_REG_MAP) || MapUtils.isEmpty(REG_PATTERN_MAP)) {
                return logStr;
            }
            //2.判断内容是否有需要脱敏的敏感信息
            Set<String> charKeys = KEY_REG_MAP.keySet();
            for (String key : charKeys) {
                if (logStr.contains(key)) {
                    String regExp = KEY_REG_MAP.get(key);
                    logStr = matchingAndEncrypt(logStr, regExp, key);
                }
            }
            return logStr;
        } catch (Exception e) {
            logger.info(">>>>>>>>> 脱敏处理异常 ERROR:{}", e);
            //如果抛出异常为了不影响流程，直接返回原信息
            return logStr;
        }
    }

    /**
     * 正则匹配对应的对象。
     *
     * @param msg
     * @param regExp
     * @return
     */
    private static String matchingAndEncrypt(String msg, String regExp, String key) {
        Pattern pattern = REG_PATTERN_MAP.get(regExp);
        if (pattern == null) {
            logger.info(">>> logger 没有匹配到对应的正则表达式 ");
            return msg;
        }
        Matcher matcher = pattern.matcher(msg);
        int length = key.length() + 5;
        boolean contains = Log4j2Rule.USER_NAME_STR.contains(key);
        String hiddenStr = "";
        while (matcher.find()) {
            String originStr = matcher.group();
            if (contains) {
                // 计算关键词和需要脱敏词的距离小于5。
                int i = msg.indexOf(originStr);
                if (i < 0) {
                    continue;
                }
                int span = i - length;
                int startIndex = span >= 0 ? span : 0;
                String substring = msg.substring(startIndex, i);
                if (StringUtils.isBlank(substring) ||  !substring.contains(key)) {
                    continue;
                }
                hiddenStr = hideMarkStr(originStr);
                msg = msg.replace(originStr, hiddenStr);
            } else {
                hiddenStr = hideMarkStr(originStr);
                msg = msg.replace(originStr, hiddenStr);
            }

        }
        return msg;
    }

    /**
     * 标记敏感文字规则
     *
     * @param needHideMark
     * @return
     */
    private static String hideMarkStr(String needHideMark) {
        if (StringUtils.isBlank(needHideMark)) {
            return "";
        }
        int startSize = 0, endSize = 0, mark = 0, length = needHideMark.length();

        StringBuffer hideRegBuffer = new StringBuffer("(\\S{");
        StringBuffer replaceSb = new StringBuffer("$1");

        if (length > 4) {
            int i = length / 3;
            startSize = i;
            endSize = i;
        } else {
            startSize = 1;
            endSize = 0;
        }

        mark = length - startSize - endSize;
        for (int i = 0; i < mark; i++) {
            replaceSb.append("*");
        }
        hideRegBuffer.append(startSize).append("})\\S*(\\S{").append(endSize).append("})");
        replaceSb.append("$2");
        needHideMark = needHideMark.replaceAll(hideRegBuffer.toString(), replaceSb.toString());
        return needHideMark;
    }


    /**
     * 创建插件
     */
    @PluginFactory
    public static Layout createLayout(@PluginAttribute(value = "pattern") final String pattern,
                                      @PluginAttribute(value = "charset") final Charset charset) {
        return new CustomPatternLayout(charset, pattern);
    }


    @Override
    public String toSerializable(LogEvent event) {
        return hideMarkLog(patternLayout.toSerializable(event));
    }

}
```

关于其中的一些细节，比如`@Plugin`、`@PluginFactory`这两个注解什么意思？log4j2如何实现自定义一个插件，这里不再详细介绍，不是本文重点，有兴趣的可以查看`log4j2`的官方文档。

**2、自定义自己的脱敏规则**

上述代码中的`Log4j2Rule`则是脱敏规则静态类，我这里是直接放在了静态类中配置，实际项目中可以设置到配置文件中，代码如下：

```java
/**
 * 现在拦截加密的日志有三类:
 * 1，身份证
 * 2，姓名
 * 3，身份证号
 * 加密的规则后续可以优化在配置文件中
 **/
public class Log4j2Rule {

    /**
     * 正则匹配 关键词 类别
     */
    public static Map<String, String> regularMap = new HashMap<>();
    /**
     * TODO  可配置
     * 此项可以后期放在配置项中
     */
    public static final String USER_NAME_STR = "Name,name,联系人,姓名";
    public static final String USER_IDCARD_STR = "empCard,idCard,身份证,证件号";
    public static final String USER_PHONE_STR = "mobile,Phone,phone,电话,手机";

    /**
     * 正则匹配，自己根据业务要求自定义
     */
    private static String IDCARD_REGEXP = "(\\d{17}[0-9Xx]|\\d{14}[0-9Xx])";
    private static String USERNAME_REGEXP = "[\\u4e00-\\u9fa5]{2,4}";
    private static String PHONE_REGEXP = "(?<!\\d)(?:(?:1[3456789]\\d{9})|(?:861[356789]\\d{9}))(?!\\d)";

    static {
        regularMap.put(USER_NAME_STR, USERNAME_REGEXP);
        regularMap.put(USER_IDCARD_STR, IDCARD_REGEXP);
        regularMap.put(USER_PHONE_STR, PHONE_REGEXP);
    }

}
```

经过上述两个步骤，自定义的`PatternLayout`已经完成，下面将是改写`log4j2.xml`这个配置文件了。



### 4. 修改log4j2.xml配置文件

其实这里修改很简单，原配置文件是直接使用`PatternLayout`进行日志格式化的，那么只需要将默认的`<PatternLayout/>`这个节点替换成`<CustomPatternLayout/>`，如下图：

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/4.png)

直接全局替换掉即可，至此，这个配置文件就修改完成了。

### 5. 演示效果

在**步骤3**这边自定义了脱敏规则静态类`Log4j2Rule`，其中定义了姓名、身份证、号码这三个脱敏规则，如下：

![](https://img.java-family.cn/Spring%20Boot%20%E6%95%B0%E6%8D%AE%E8%84%B1%E6%95%8F/5.png)

下面就来演示这三个规则能否正确脱敏，直接使用日志打印，代码如下：

```java
@Test
public void test3(){
	log.debug("身份证：{}，姓名：{}，电话：{}","320829112334566767","不才陈某","19896327106");
}
```

控制台打印的日志如下：

```java
身份证：320829******566767，姓名：不***，电话：198*****106
```

哦豁，成功了，so easy！！！



### 6. 总结

日志脱敏的方案很多，陈某也只是介绍一种常用的，有兴趣的可以研究一下。



## 五、总结

本篇文章从三个维度介绍了隐私数据的脱敏实现方案，码字不易，赶紧点赞收藏吧！！！

> 源码已经上传GitHub，需要的公众号码猿技术专栏，回复关键词`数据脱敏`获取。

