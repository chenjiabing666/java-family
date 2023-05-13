**大家好，我是不才陈某~**

这是[《ShardingSphere 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2389616635193393153#wechat_redirect)专栏的第**2**篇文章，往期文章如下：

- [聊聊 Sharding-JDBC 分库分表](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511195&idx=1&sn=4783b892ef74bbe93e62603a204741ee&chksm=fcf77356cb80fa40d6d1335c4829fe590d23509e7bd4360c31f5fe661e8106403ca9d95770b1&token=719297142&lang=zh_CN#rd)

安全控制一直是治理的重要环节，数据脱敏属于安全控制的范畴。对互联网公司、传统行业来说，数据安全一直是极为重视和敏感的话题。

数据脱敏是指对某些敏感信息通过脱敏规则进行数据的变形，实现敏感隐私数据的可靠保护。

涉及客户安全数据或者一些商业性敏感数据，如**身份证号**、**手机号**、**卡号**、**客户号**等个人信息按照相关部门规定，都需要进行数据脱敏。

关于数据脱敏陈某前面也发表过两篇文章，如下：

- [大厂也在用的 6种 数据脱敏方案，别做泄密内鬼！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247504633&idx=1&sn=85f0e78d990a0fc6d3c1135dd3b49b4a&chksm=fcf70d34cb80842246fe7623cc5dba23ed316a33b9447f1ff6e71367f941e0891f6e815c7c0f&token=201261640&lang=zh_CN#rd)：介绍了常用的脱敏方案
- [Springboot 日志、配置文件、接口数据如何脱敏？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247495798&idx=1&sn=95f537d8e82823b668b1c7a2d6842827&chksm=fcf72fbbcb80a6ad42a12221d3b075670deb5014389db03b7cb8a9444f3776c54d52b98ba404&token=201261640&lang=zh_CN#rd)：介绍了SpringBoot中的日志、配置文件、接口数据的脱敏

今天来深入聊一下 Sharding-JDBC 如何对敏感数据脱敏，仅在持久层脱敏。



## 几个重要概念

Sharding-JDBC 在底层进行了脱敏的封装，让开发人员在无感知的情况下进行数据脱敏，来看一下官方的详情图，如下：

![](https://java-family.cn/BlogImage/ShardingSphere/29.png)

下面针对上图中涉及到的几个名词进行详细的解释，在下文实战中做个铺垫。



**1.数据源配置**

这个就是Datasource的配置，这个在上篇文章中也配置过



**2.加密器配置**

加密器就涉及到数据脱敏了，Sharding-JDBC 内置了两个加密器，如下：

1. **MD5Encryptor**：MD5加密算法，一种不可逆的加密方式，通常用来对密码进行加密
2. **AESEncryptor**：AES加密算法，一种可逆的加密方式，通常用来对回显的字段加密，比如身份证、手机号码

Sharding-JDBC还支持自定义加密器，这个会在下文介绍。



**3.脱敏表配置**

用于告诉 Sharding-JDBC 数据表里哪个列用于存储密文数据（**cipherColumn**）、哪个列用于存储明文数据（**plainColumn**）以及用户想使用哪个列进行SQL编写（**logicColumn**）：

- **logicColumn**：逻辑列，这个和前文中逻辑表类似，用于实际的SQL编写，比如数据库中真实字段是**cipher_pwd**，但是在Sharding-JDBC配置时指定逻辑列的名称为：**pwd**，那么在写SQL的时候就要使用逻辑列**pwd**进行查询。
- **cipherColumn**：存储密文数据的字段
- **plainColumn**：存储明文数据的字段，一般不使用，不然脱敏也毫无意义



**4.查询属性的配置**

当底层数据库表里同时存储了明文数据、密文数据后，该属性开关用于决定是直接查询数据库表里的明文数据进行返回，还是查询密文数据通过Encrypt-JDBC解密后返回。



## 数据脱敏实战

基本概念介绍完了，下面就使用Sharding-JDBC进行数据脱敏。

这里就不再演示分库分表了，直接用单库进行脱敏演示。

> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9534**  获取！

### 1. 新建表

笔者这里新建了一张用户表t_user，如下：

```sql
CREATE TABLE `t_user` (
  `user_id` bigint(20) NOT NULL COMMENT '用户唯一ID',
  `fullname` varchar(50) DEFAULT NULL COMMENT '名字',
  `user_type` varchar(255) DEFAULT NULL COMMENT '类型',
  `cipher_pwd` varchar(255) DEFAULT NULL COMMENT '密码',
  `mobile` varchar(100) DEFAULT NULL COMMENT '手机号',
  `mobile_data` varchar(100) DEFAULT NULL COMMENT '手机号',
  `id_card` varchar(60) DEFAULT NULL COMMENT '身份证',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```



### 2. 数据源配置

数据源这里使用单数据源，配置很简单，上篇文章也演示过，配置如下：

```yaml
spring:
  # Sharding-JDBC的配置
  shardingsphere:
    datasource:
      names: ds
     # 数据源ds配置
      ds:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost:3306/user_db?useUnicode=true&characterEncoding=utf-8
        username: root
        password: 123456
```



### 3. 加密器声明

需要用到什么加密器需要事先在配置文件中声明，这样才能在字段中去引用，配置如下：

```yaml
spring:
    encrypt:
      encryptors:
        # md5加密算法-sharding-jdbc内置的算法，这里名称任意
        encryptor_md5:
         # 别名，这里一定要是MD5
          type: MD5
        # aes加密算法-sharding-jdbc内置的算法，这里名称任意
        encryptor_aes:
        # 别名，这里一定要是aes
          type: aes
          props:
            # 设置秘钥
            aes.key.value: myjszl
```

上述总计配置了两种Sharding-JDBC内置的加密器，如下：

1. **encryptor_md5**：MD5Encryptor加密器，这里的名称可以任意，但是**type**这个属性一定要是MD5
2. **encryptor_aes**：AESEncryptor加密器，对称加密算法，因此需要指定加密的秘钥：**aes.key.value**

Sharding-JDBC指定规则如下：

```properties
#加解密器类型，可自定义或选择内置类型：MD5/AES 
spring.shardingsphere.encrypt.encryptors.<encryptor-name>.type= 

#属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value
spring.shardingsphere.encrypt.encryptors.<encryptor-name>.props.<property-name>= 
```

>  关于如何自定义加密器将在下文介绍。

### 4. 对数据脱敏配置

下面针对三个字段进行脱敏，如下：

- **cipher_pwd**：密码使用不可逆的加密器MD5Encryptor
- **id_card**：身份证使用可逆的加密器AESEncryptor
- **mobile**：手机号使用可逆的加密器AESEncryptor

详细的配置如下：

```yaml
spring:
  # Sharding-JDBC的配置
  shardingsphere:
    encrypt:
      tables:
        t_user:
          columns:
            # 逻辑列，sharding-jdbc中写SQL需要用到的列
            password:
              # 存储明文的字段
              #plainColumn: password
              # 存储密文的字段
              cipherColumn: cipher_pwd
              # 指定加密器
              encryptor: encryptor_md5
            # 身份证号的逻辑列，使用aes这种可逆的加密算法
            id_card:
              cipherColumn: id_card
              encryptor: encryptor_aes
            # 手机号的逻辑列，使用aes这种可逆的加密算法
            mobile:
              cipherColumn: mobile
              encryptor: encryptor_aes
```



Sharding-JDBC 指定的规则如下：

```properties
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.encryptor= #加密器名字

spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.plainColumn= #存储明文的字段

spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.cipherColumn= #存储密文的字段
```



**注意**：上述配置中的密码这个字段，数据库表中的真实字段是**cipher_pwd**，但是这里笔者指定的逻辑列是password，因此在写SQL的时候，一定要写**password**这个逻辑列，比如查询的SQL，如下：

```sql
SELECT
	password AS cipherPwd,
	fullname,
	user_type,
	id_card AS id_card
FROM
	t_user where user_id=?
```



现在向其中插入几条数据看看效果，单元测试如下：

```java
@Test
public void testInsertUser() {
    for (int i = 0; i < 10; i++) {
        User user = new User();
        user.setFullName("不才陈某");
        user.setCipherPwd("abc123");
        user.setIdCard("320829198708012232");
        user.setUserId((long)i);
        user.setMobile("13852331509");
        userMapper.insertUser(user);
    }
}
```

数据如下：

![](https://java-family.cn/BlogImage/ShardingSphere/30.png)

可以看到数据持久化到数据库中已经脱敏了。

问题来了：**那么查询的效果出来的效果是什么？**

试想一下，MD5加密器是不可逆的，AES加密器是可逆的，那么符合正常逻辑的状态下就应该是密码这个字段查询出来的还是密文（不可逆），身份证、手机号查询出来的应该是明文。

新建查询的单元测试，如下：

```java
@Test
public void testList() throws JsonProcessingException {
    List<User> users = userMapper.listAll();
    ObjectMapper objectMapper = new ObjectMapper();
    String json = objectMapper.writeValueAsString(users);
    System.out.println(json);
}
```

结果如下：

![](https://java-family.cn/BlogImage/ShardingSphere/31.png)

**很清楚了，一切都朝着我们的预料的方向发展~**

> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9534**  获取！



## 限制条件

字段进行脱敏后在写SQL时有一些操作是不支持的，如下：

1. 脱敏字段无法支持比较操作，如：`大于小于`、`ORDER BY`、`BETWEEN`、`LIKE`等。
2. 脱敏字段无法支持计算操作，如：`AVG`、`SUM`以及计算表达式 。

## 原理

其实Sharding-JDBC数据脱敏原理很简单，看一下官方给的一张图：

![](https://java-family.cn/BlogImage/ShardingSphere/32.png)



### 1.  插入数据

加密器有一个公共的接口**Encryptor**，如下：

```java
public interface Encryptor extends TypeBasedSPI {
    
    /**
     * Initialize.
     */
    void init();
    
    /**
     * Encode.
     * 
     * @param plaintext plaintext
     * @return ciphertext
     */
    String encrypt(Object plaintext);
    
    /**
     * Decode.
     * 
     * @param ciphertext ciphertext
     * @return plaintext
     */
    Object decrypt(String ciphertext);
}
```



当插入数据涉及到加密字段，即是定义的逻辑列，那么Sharding-JDBC内部会将这条SQL改写，将逻辑列替换成表的真实列，并且调用 `encrypt(Object plaintext)`方法将明文加密成密文后存储进去。

> 具体的源码就不贴出来了，后面源码章节介绍



### 2.  更新数据

更新和插入数据一样，同样会将SQL改写，调用`encrypt(Object plaintext)`方法将明文加密成密文后存储进去。



### 3.  查询数据

查询数据就比较复杂了，这里只讨论默认情况，则是使用加密列查询，默认配置如下：

```properties
spring.shardingsphere.props.query.with.cipher.column=true
```

查询分为两类，如下：



**1.where条件中不带脱敏逻辑列**

这种情况也就是在查询结果集中涉及到脱敏的逻辑列，但是在查询条件中不涉及，那么在返回结果的时候则会调用加密器的解密方法`Object decrypt(String ciphertext)`去将结果解密返回



**2.where条件中带脱敏逻辑列**

这种情况就比较复杂了，where条件中涉及了脱敏逻辑列，那么在改写SQL时会调用加密器的加密方法`String encrypt(Object plaintext)`将其加密成密文去查询；

同样返回结果集也会调用加密器的解密方法`Object decrypt(String ciphertext)`去将结果解密返回



## 加密策略

Sharding-JDBC默认提供了两种内置的加密器，但是实际开发中这两种肯定是不够用的，需要开发人员去自定义加密器应该各种场景。

Sharding-JDBC提供了两类加密策略接口，如下：

### **1. Encryptor**

提供`encrypt()`, `decrypt()`两种方法对需要脱敏的数据进行加解密；在用户进行`INSERT`, `DELETE`, `UPDATE`时，ShardingSphere会按照用户配置，对SQL进行解析、改写、路由，并会调用`encrypt()`将数据加密后存储到数据库, 而在`SELECT`时，则调用`decrypt()`方法将从数据库中取出的脱敏数据进行逆向解密，最终将原始数据返回给用户。

接口如下：

```java
public interface Encryptor extends TypeBasedSPI {
    void init();
    String encrypt(Object plaintext);
    Object decrypt(String ciphertext);
}
```



### 2.  QueryAssistedEncryptor

相比较于第一种脱敏方案，该方案更为安全和复杂。它的理念是：即使是相同的数据，如两个用户的密码相同，它们在数据库里存储的脱敏数据也应当是不一样的。这种理念更有利于保护用户信息，防止撞库成功。

它提供三种函数进行实现，分别是`encrypt()`, `decrypt()`, `queryAssistedEncrypt()`。在`encrypt()`阶段，用户通过设置某个**变动种子**，例如**时间戳**。针对原始数据+变动种子组合的内容进行加密，就能保证即使原始数据相同，也因为有变动种子的存在，致使加密后的脱敏数据是不一样的。在`decrypt()`可依据之前规定的加密算法，利用种子数据进行解密。

虽然这种方式确实可以增加数据的保密性，但是另一个问题却随之出现：相同的数据在数据库里存储的内容是不一样的，那么当用户按照这个加密列进行等值查询(`SELECT FROM table WHERE encryptedColumnn = ?`)时会发现无法将所有相同的原始数据查询出来。

为此，我们提出了辅助查询列的概念。该辅助查询列通过`queryAssistedEncrypt()`生成，与`decrypt()`不同的是，该方法通过对原始数据进行另一种方式的加密，但是针对原始数据相同的数据，这种加密方式产生的加密数据是一致的。

将`queryAssistedEncrypt()`后的数据存储到数据中用于辅助查询真实数据。因此，数据库表中多出这一个辅助查询列。

由于`queryAssistedEncrypt()`和`encrypt()`产生不同加密数据进行存储，而`decrypt()`可逆，`queryAssistedEncrypt()`不可逆。 在查询原始数据的时候，会自动对SQL进行解析、改写、路由，利用辅助查询列进行 `WHERE`条件的查询，却利用 `decrypt()`对`encrypt()`加密后的数据进行解密，并将原始数据返回给用户。这一切都是对用户透明化的。

> 上文部分内容摘抄自官方文档

**简单概括一下**：

1. **QueryAssistedEncryptor**更加安全，加了一个变动因子一起加密，这样即使内容一样加密后的密文也是不同的
2. 为了查询方便，提供了一个辅助查询列，这个辅助查询列中的数据不带变动因子，直接明文加密的，可以根据辅助查询列查询
3. 这一切的操作都是透明的，开发人员无须关心，只需要按照给定的规则配置

接口如下：

```java
public interface QueryAssistedEncryptor extends Encryptor {
    String queryAssistedEncrypt(String plaintext);
}
```



> QueryAssistedEncryptor这类加密策略并无内置的加密器，需要开发人员自定义实现



## 如何自定义加密器

上文介绍到了Sharding-JDBC支持的两种加密策略，肯定都是要实现一下，下面将会针对两种策略去介绍一下如何自定义。

**前提**：由于Sharding-JDBC中的加密器是使用**SPI**方式让开发人员扩展的，因此你还要了解一下**SPI**，有不清楚的可以看我之前的文章：[聊聊 Java SPI 机制](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511021&idx=1&sn=2a1b394387970210fe449193ca6535d7&chksm=fcf77420cb80fd367124c35cae6b0150cd2e75c9f26fee583f921f335168d659f971df0dec59&token=201261640&lang=zh_CN#rd)



### 1. Encryptor 自定义实现

自定义很简单，直接实现Encryptor 接口即可，重写其中的加密、解密方法。

下面自定义一个**SHA256**加密算法器，这是一种不可逆的算法，如下：

```java
/**
 * @author 不才陈某 公众号：码猿技术专栏
 * 自定义的加密解密算法，基于sha256
 */
@Data
public class Sha256HexEncryptor implements Encryptor {

    /**
     * 别名，配置时需要
     */
    public final static String ALGORITHM_NAME="SHA256";

    private Properties properties = new Properties();

    @Override
    public void init() {

    }

    /**
     * 加密
     * INSERT, DELETE, UPDATE时会调用该方法进行加密存储到数据库中
     * @param plaintext 明文
     * @return  加密后的密文
     */
    @Override
    public String encrypt(final Object plaintext) {
        if (null == plaintext) {
            return null;
        }
        return DigestUtils.sha256Hex(String.valueOf(plaintext));
    }

    /**
     * 解密
     * 在SELECT 查询会调用该方法进行解密
     * @param ciphertext 密文
     * @return 由于sha256是一种不可逆的算法，因此直接返回密文
     */
    @Override
    public String decrypt(final String ciphertext) {
        return ciphertext;
    }

    /**
     * 别名，在配置中指定的名称
     */
    @Override
    public String getType() {
        return Sha256HexEncryptor.ALGORITHM_NAME;
    }
}
```

加密解密的过程和**MD5Encryptor**加密器类似，不再详细介绍了，很容易理解。

由于是使用SPI的方式，因此还需要在**resource/META-INF/services**目录中新建一个**org.apache.shardingsphere.encrypt.strategy.spi.Encryptor**文件，内容如下：

```properties
com.java.family.shardingjdbc003.encryptor.Sha256HexEncryptor
```

好了，现在这个SHA256加密器就已经定义好了，可以在配置文件中配置了。

现在使用自定义SHA256加密器对密码这个列进行加密，分为如下步骤：

**1.声明加密器**

声明很简单，配置如下：

```yaml
spring:
  # Sharding-JDBC的配置
  shardingsphere:
    encrypt:
      encryptors:
        # sha256加密算法，自定义的
        encryptor_sha256:
          type: SHA256
```



**2.逻辑列配置加密器**

只需要将加密器的名字改成上面声明的**encryptor_sha256**即可，配置如下：

```yaml
spring:
  # Sharding-JDBC的配置
  shardingsphere:
    encrypt:
      tables:
        t_user:
          columns:
            # 逻辑列，sharding-jdbc中写SQL需要用到的列
            password:
              # 存储明文的字段
              #plainColumn: password
              # 存储密文的字段
              cipherColumn: cipher_pwd
              # 指定加密器
              encryptor: encryptor_sha256
```

至此，配置成功了，自己去演示看一下吧，这里就不再细说了。



### 2. QueryAssistedEncryptor 自定义实现

下面通过Base64加密算法自定义实现一个QueryAssistedEncryptor加密器，如下：

```java
/**
 * @author 不才陈某 公众号：码猿技术专栏
 * 自定义QueryAssistedEncryptor加密器
 */
@Data
public class Base64AssistedEncryptor implements QueryAssistedEncryptor {

    /**
     * 别名，配置时需要
     */
    public final static String ALGORITHM_NAME="Base64_Assisted";

    private Properties properties = new Properties();

    /**
     * 辅助查询列的加密方法
     */
    @Override
    public String queryAssistedEncrypt(String plaintext) {
        if (null == plaintext) {
            return null;
        }
        return Base64.encode(plaintext);
    }

    @Override
    public void init() {

    }

    /**
     * 加密方法
     * 使用时间戳作为变动因子
     */
    @Override
    public String encrypt(final Object plaintext) {
        if (null == plaintext) {
            return null;
        }
        //获取时间戳作为变动因子
        String randomFactor =String.valueOf( new Date().getTime());
        return Base64.encode(plaintext+"_"+randomFactor);
    }

    /**
     * 解密方法
     * Base64是一个可逆的加密算法，因此可以对密文进行解密并且剔除变动因子则为明文
     */
    @SneakyThrows
    @Override
    public Object decrypt(final String ciphertext) {
        if (null == ciphertext) {
            return null;
        }
        return new String(Base64.decode(ciphertext),"UTF-8").split("_")[0];
    }

    @Override
    public String getType() {
        return ALGORITHM_NAME;
    }
}
```

需要注意以下两点：

- `queryAssistedEncrypt()`：该方法在插入、更新逻辑列设置辅助查询列值、逻辑列作为where查询条件时会被调用对辅助查询列加密
- `decrypt()`：这里的Base64是可逆的加密算法，因此只需要对其解密，并且剔除变动因子则为明文

同样的也需要在**resource/META-INF/services**目录中新建一个**org.apache.shardingsphere.encrypt.strategy.spi.Encryptor**文件，内容如下：

```properties
com.java.family.shardingjdbc003.encryptor.Base64AssistedEncryptor
```

配置也很简单，同样分为两步，如下：

**1.声明加密器**

配置如下：

```yaml
spring:
  # Sharding-JDBC的配置
  shardingsphere:
    encrypt:
      encryptors:
        # Base64加密算法，自定义的
        encryptor_base64_assisted:
          type: Base64_Assisted
```



**2.逻辑列配置加密器**

只需要将加密器的名字改成上面声明的**encryptor_base64_assisted**即可，配置如下：

```yaml
![33](https://java-family.cn/BlogImage/ShardingSphere/33.png)spring:
  # Sharding-JDBC的配置
  shardingsphere:
    encrypt:
      tables:
        t_user:
          columns:
            # 逻辑列，sharding-jdbc中写SQL需要用到的列
            mobile:
              # 密文存储的列
              cipherColumn: mobile
              # 辅助查询列，表中真实的字段名
              assistedQueryColumn: mobile_data
              encryptor: encryptor_base64_assisted
```

唯一不同的就是多了一个**assistedQueryColumn**辅助查询列的配置。

好了，上述配置好了以后就可以进行单元测试了，插入数据结果：

![](https://java-family.cn/BlogImage/ShardingSphere/33.png)

可以看到**mobile**这个字段的值都是不同的，但是**mobile_data**这个辅助查询列都是相同的，因为辅助查询列并未使用变动因子进行加密。

关于查询如果涉及到`mobile`的条件查询，那么将会调用`queryAssistedEncrypt()`方法加密后根据辅助查询`mobile_data`列进行查询，SQL如下图：

![](https://java-family.cn/BlogImage/ShardingSphere/34.png)



> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9534**  获取！



## 总结

本文介绍了如何利用Sharding-JDBC 进行数据脱敏以及如何自定义加密器，文中涉及的案例代码都已经提交GitHub。

> 源码已经上传GitHub，关注公众号：**码猿技术专栏**，回复关键词：**9534**  获取！

## 最后说一句（别白嫖，求关注）

陈某每一篇文章都是精心输出，已经写了**3个专栏**，整理成**PDF**，获取方式如下：

1. [《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Cloud 进阶** 获取！
2. [《Spring Boot 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1532834475389288449#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Spring Boot进阶** 获取！
3. [《Mybatis 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=1500819225232343046#wechat_redirect)PDF：关注公众号：【**码猿技术专栏**】回复关键词 **Mybatis 进阶** 获取！

> 另外陈某的[《Spring Cloud Alibaba 实战》](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247511213&idx=1&sn=9635f9d57390269edef798fa66a03e3b&chksm=fcf77360cb80fa76fe5c9abee83c54fb4e06038d555663c07f5c1be7838bc7abdee5417a8e18&token=719297142&lang=zh_CN#rd)**视频专栏**已经杀青了，需要**订阅**的点击左下角**阅读原文**。
