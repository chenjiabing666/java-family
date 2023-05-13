## 前言

- 作为一个资深后端码农天天都要和数据库打交道，最早使用的是 Hiberate，一个封装性极强的持久性框架。自从接触到 Mybatis 就被它的灵活性所折服了，可以自己写 SQL，虽然轻量级，但是麻雀虽小，五脏俱全。这篇文章就来讲讲什么是 Mybatis，如何简单的使用 Mybatis。

## 什么是 Mybatis

- MyBatis 是一款优秀的`持久层`框架，它支持`自定义 SQL`、`存储过程`以及`高级映射`。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 `XML` 或注解来配置和映射原始类型、接口和 `Java POJO`（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

## 环境搭建

- 本篇文章使用的环境是`SpringBoot+Mybatis+Mysql`

### Maven 依赖

- MySQL 驱动依赖和 Druid 连接池的依赖

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.40</version>
            <scope>runtime</scope>
        </dependency>

        <!--druid连接池-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.9</version>
        </dependency>
```

- Mybatis 启动包依赖，此处导入的是 SpringBoot 和 Mybatis 整合启动器的依赖，点击去可以看到，这个启动包依赖了`mybatis`和`mybatis-spring`（Mybatis 和 Spring 整合的 Jar 包），因此使用 SpringBoot 之后只需要导入这个启动器的依赖即可。

```xml
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>
```

- 以上两个依赖添加成功后，Maven 环境就已经配置完了。

### 数据库连接池配置（Druid）

- 这个不是本文的重点，而且网上很多教程，我就简单的配置一下，在 SpringBoot 的`application.properties`中配置即可。

```properties
##单一数据源
spring.datasource.url=jdbc\:mysql\://127.0.0.1\:3306/vivachekcloud_pzhdermyy?useUnicode\=true&characterEncoding\=UTF-8&zeroDateTimeBehavior\=convertToNull&useSSL\=false
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
#初始化连接大小
spring.datasource.druid.initial-size=0
#连接池最大使用连接数量
spring.datasource.druid.max-active=20
#连接池最小空闲
spring.datasource.druid.min-idle=0
#获取连接最大等待时间
spring.datasource.druid.max-wait=6000
spring.datasource.druid.validation-query=SELECT 1
#spring.datasource.druid.validation-query-timeout=6000
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
spring.datasource.druid.test-while-idle=true
#配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
spring.datasource.druid.time-between-eviction-runs-millis=60000
#置一个连接在池中最小生存的时间，单位是毫秒
spring.datasource.druid.min-evictable-idle-time-millis=25200000
#spring.datasource.druid.max-evictable-idle-time-millis=
#打开removeAbandoned功能,多少时间内必须关闭连接
spring.datasource.druid.removeAbandoned=true
#1800秒，也就是30分钟
spring.datasource.druid.remove-abandoned-timeout=1800
#<!-- 1800秒，也就是30分钟 -->
spring.datasource.druid.log-abandoned=true
spring.datasource.druid.filters=mergeStat
#spring.datasource.druid.verifyServerCertificate
#spring.datasource.filters=stat,wall,log4j
# 通过connectProperties属性来打开mergeSql功能；慢SQL记录
spring.datasource.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
```

## 基础概念

- `dao`层：用于存放和数据库交互的文件，Mybatis 的`interface`都放在此层
- `service`层：用于存放业务逻辑的文件。

## 配置 xml 文件存放的位置

- Mybatis 中`xml`的文件默认是要和`interface`放在一个包下的，并且文件的名称要一样。
- 在和 SpringBoot 整合后有两种配置方式，下面详细介绍。

### application.properties 中设置

- 既然是和 SpringBoot 整合，那么万变不离`xxxAutoConfiguration`这个配置类了，Mybatis 的配置类就是`MybatisAutoConfiguration`，如下：

```java
@org.springframework.context.annotation.Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration implements InitializingBean {}
```

- 可以看到`@EnableConfigurationProperties(MybatisProperties.class)`这行代码，就是将 properties 中的属性映射到 MybatisProperties 这个成员属性中，因此设置的方式就要看其中的属性。

```java
public class MybatisProperties {
  //前缀
  public static final String MYBATIS_PREFIX = "mybatis";

  /**
   * Mybatis配置文件的位置
   */
  private String configLocation;

  /**
   * Mybatis的Mapper的xml文件的位置
   */
  private String[] mapperLocations;
```

- 因此设置的方式很简单，如下：

```properties
## xml文件放置在/src/main/resource/mapper/文件夹下
mybatis.mapper-locations=classpath*:/mapper/**/*.xml
```

### 配置类中设置

- 不是本章重点，后面在讲 Mybatis 和 SpringBoot 整合的文章会涉及到该内容。

## 配置扫描 Mybatis 的 interface

- 在和 SpringBoot 整合后，扫描 Mybatis 的接口，生成代理对象是一件很简单的事，只需要一个注解即可。

### @Mapper

- 该注解标注在 Mybatis 的`interface`类上，SpringBoot 启动之后会扫描后会自动生成代理对象。实例如下：

```java
@Mapper
public interface UserInfoMapper {

    int insert(UserInfo record);

    int insertSelective(UserInfo record);
    }
```

- 缺点：每个`interface`都要标注一个，很鸡肋，一个项目中的 interface 少说也有上百个吧。

### @MapperScan

- `@Mapper`注解的升级版，标注在配置类上，用于一键扫描 Mybatis 的`interface`。
- 使用也是很简单的，直接指定接口所在的包即可，如下：

```java
@MapperScan({"com.xxx.dao"})
public class ApiApplication {}
```

- `@MapperScan`和`@Mapper`这两个注解千万不要重复使用。
- 优点：一键扫描，不用每个 interface 配置。

## 基本的 crud

- 既然和数据库交互，避免不了 crud 操作，就安心做一个妥妥的`crud boy`吧。
- 针对 Mybatis 其实有两套方法映射，一个是 XML 文件的方式，一个是注解的方式。但是今天只讲 XML 文件的方式，原因很简单，注解的方式企业不用，谁用谁倒霉，哈哈。

### 查询

- 查询语句是 MyBatis 中最常用的元素之一——光能把数据存到数据库中价值并不大，还要能重新取出来才有用，多数应用也都是查询比修改要频繁。 MyBatis 的基本原则之一是：在每个插入、更新或删除操作之间，通常会执行多个查询操作。因此，MyBatis 在查询和结果映射做了相当多的改进。一个简单查询的 select 元素是非常简单的。

```xml
<select id="selectPersonById" parameterType="int" resultType="com.myjszl.domain.Person">
  SELECT name,age,id FROM PERSON WHERE ID = #{id}
</select>
```

- 对应的`interface`的方法如下：

```java
Person selectPersonById(int id);
```

- `<select>`这个标签有很多属性，比较常用的属性如下：

  - `id`（必填）：在命名空间中唯一的标识符，可以被用来引用这条语句。和`interface`中的`方法名`要一致。
  - `parameterType`（可选）：将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。
  - `resultType`：期望从这条语句中返回结果的类全限定名或别名。 注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 `resultType` 和 `resultMap` 之间只能同时使用一个。
  - `resultMap`：对外部 `resultMap` 的命名引用。结果映射是 MyBatis 最强大的特性，如果你对其理解透彻，许多复杂的映射问题都能迎刃而解。 `resultType` 和 `resultMap` 之间只能同时使用一个。

### 变更

- 数据变更语句 insert，update 和 delete 的实现非常接近。
- 下面是 insert，update 和 delete 语句的示例：

```xml
<insert id="insertAuthor">
  insert into Author (id,username,password,email,bio)
  values (#{id},#{username},#{password},#{email},#{bio})
</insert>

<update id="updateAuthor">
  update Author set
    username = #{username},
    password = #{password},
    email = #{email},
    bio = #{bio}
  where id = #{id}
</update>

<delete id="deleteAuthor">
  delete from Author where id = #{id}
</delete>
```

## #{}和\${}的区别

- 上面的例子中我们可以看到使用的都是`#{}`，关于`#{}`和`${}`的区别也是在很多初级工程师的面试最常被问到的，现在只需要记住区别就是`#{}`使用了 JDBC 的预编译，可以防止 SQL 注入，提高了安全性，`${}`并没有预编译，安全性不够。在后面 Mybatis 的源码讲解中将会涉及到为什么一个用了预编译，一个没用。

## 自增 ID 的返回

- 关于 Mysql 的文章中有提到，设计一个表最好要有一个自增 ID，无论这个 ID 你是否用到，具体原因不在解释，可以翻看之前的文章。
- 有了自增 ID，插入之后并不能自动返回，但是我们又需要这个 ID 值，那么如何返回呢？
- `<insert>`标签提供了两个属性用来解决这个问题，如下：
  - `useGeneratedKeys`：设置为 true，表示使用自增主键返回
  - `keyProperty`：指定返回的自增主键映射到`parameterType`的哪个属性中。
- 假设插入`Person`，并且 person 表中的自增主键 id 需要返回，XML 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xxx.dao.PersonMapper">
  <insert id='addPerson' parameterType='com.xxx.domain.Person' useGeneratedKeys="true"
    keyProperty="id" >
    insert into person(name,age)
    values(#{name},#{age});
  </insert>
</mapper>
```

## SQL 代码片段

- 这个元素可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用。 参数可以静态地（在加载的时候）确定下来，并且可以在不同的 include 元素中定义不同的参数值。比如：

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```

- 这个 SQL 片段可以在其它语句中使用，例如：

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

## 开启驼峰映射

- DBA 在设计数据库的时候，往往使用的是下划线(`_`)的方式，比如`user_id`。但是 Java 是不规范的，我们通常将它转换为`userId`，这就是驼峰命名方法。
- 但是在使用 Mybatis 查询的时候，比如：

```xml
<select id='selectById' resultType='com.xxx.doamin.User'>
  select user_id from user_info
</select>
```

- 上面的`user_id`和`User`中的`userId`根本不对应，也就映射不进去，此时查询的结果就是 userId 是 null，当然我们可以使用别名的方式，SQL 可以改写为`select user_id as userId from user_info`
- 另外一种方式是不用别名，直接开启 Mybatis 的驼峰映射规则，会自动映射，开启的方式很简单，就是在`application.properties`文件配置一下，如下：

```properties
mybatis.configuration.map-underscore-to-camel-case=true
```

## 总结

- 本文主要讲了 Mybatis 与 SpringBoot 的整合过程，基本的 crud，各种标签的属性等内容，属于一个入门级别的教程，后续的内容会逐渐深入。
