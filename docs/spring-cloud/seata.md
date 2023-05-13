# 阿里分布式事务组件Seata
## 什么是Seata？

上面讲了这么多的分布式事务的理论知识，都没看到一个落地的实现，这不是吹牛逼吗？

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 **AT**、**TCC**、**SAGA** 和 **XA** 事务模式，为用户打造一站式的分布式解决方案。

- 对业务无侵入：即减少技术架构上的微服务化所带来的分布式事务问题对业务的侵入
- 高性能：减少分布式事务解决方案所带来的性能消耗

官方文档：https://seata.io/zh-cn/index.html

seata的几种术语：

- **TC（Transaction Coordinator）**：事务协调者。管理全局的分支事务的状态，用于全局性事务的提交和回滚。
- **TM（Transaction Manager）**：事务管理者。用于开启、提交或回滚事务。
- **RM（Resource Manager）**：资源管理器。用于分支事务上的资源管理，向 **TC** 注册分支事务，上报分支事务的状态，接收 **TC** 的命令来提交或者回滚分支事务。

## AT模式

seata目前支持多种事务模式，分别有**AT**、**TCC**、**SAGA** 和 **XA** ，文章篇幅有限，今天只讲常用的AT模式。

AT模式的特点就是对**业务无入侵式**，整体机制分**二阶段提交**（2PC）

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：
  1. 提交异步化，非常快速地完成
  2. 回滚通过一阶段的回滚日志进行反向补偿。

在 AT 模式下，用户只需关注自己的**业务SQL**，用户的**业务SQL** 作为一阶段，Seata 框架会自动生成事务的二阶段提交和回滚操作。

![](https://img.java-family.cn/seata/28.png)

一个典型的分布式事务过程：

- TM 向 TC 申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的 XID；
- XID 在微服务调用链路的上下文中传播；
- RM 向 TC 注册分支事务，将其纳入 XID 对应全局事务的管辖；
- TM 向 TC 发起针对 XID 的全局提交或回滚决议；
- TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求。

## 搭建Seata TC协调者

seata的协调者其实就是阿里开源的一个服务，我们只需要下载并且启动它。

下载地址：http://seata.io/zh-cn/blog/download.html

> 陈某下载的版本是`1.3.0 `，各位最好和我版本一致，这样不会出现莫名的BUG。

下载完成后，直接解压即可。但是此时还不能直接运行，还需要做一些配置。

### 创建TC所需要的表

TC运行需要将事务的信息保存在数据库，因此需要创建一些表，找到seata-1.3.0源码的`script\server\db`这个目录，将会看到以下SQL文件：

![](https://img.java-family.cn/seata/29.png)

陈某使用的是Mysql数据库，因此直接运行mysql.sql这个文件中的sql语句，创建的三张表如下图：

![](https://img.java-family.cn/seata/30.png)

### 修改TC的注册中心

找到`seata-server-1.3.0\seata\conf`这个目录，其中有一个`registry.conf`文件，其中配置了TC的注册中心和配置中心。

默认的注册中心是`file`形式，实际使用中肯定不能使用，需要改成Nacos形式，改动的地方如下图：

![](https://img.java-family.cn/seata/31.png)

需要改动的地方如下：

- type：改成nacos，表示使用nacos作为注册中心
- application：服务的名称
- serverAddr：nacos的地址
- group：分组
- namespace：命名空间
- username：用户名
- password：密码

> 最后这份文件都会放在项目源码的根目录下，源码下载方式见文末

### 修改TC的配置中心

TC的配置中心默认使用的也是`file`形式，当然要是用nacos作为配置中心了。

直接修改`registry.conf`文件，需要改动的地方如下图：

![](https://img.java-family.cn/seata/32.png)

需要改动的地方如下：

- type：改成nacos，表示使用nacos作为配置中心
- serverAddr：nacos的地址
- group：分组
- namespace：命名空间
- username：用户名
- password：密码

上述配置修改好之后，在TC启动的时候将会自动读取nacos的配置。

那么问题来了：**TC需要存储到Nacos中的配置都哪些，如何推送过去？**

在`seata-1.3.0\script\config-center`中有一个`config.txt`文件，其中就是TC所需要的全部配置。

在`seata-1.3.0\script\config-center\nacos`中有一个脚本`nacos-config.sh`则是将config.txt中的全部配置自动推送到nacos中，运行下面命令（windows可以使用git bash运行）：

```shell
# -h 主机，你可以使用localhost，-p 端口号 你可以使用8848，-t 命名空间ID，-u 用户名，-p 密码
$ sh nacos-config.sh -h 127.0.0.1 -p 8080 -g SEATA_GROUP -t 7a7581ef-433d-46f3-93f9-5fdc18239c65 -u nacos -w nacos
```

推送成功则可以在Nacos中查询到所有的配置，如下图：

![](https://img.java-family.cn/seata/33.png)



### 修改TC的数据库连接信息

TC是需要使用数据库存储事务信息的，那么如何修改相关配置呢？

上一节的内容已经将所有的配置信息都推送到了Nacos中，TC启动时会从Nacos中读取，因此我们修改也需要在Nacos中修改。

需要修改的配置如下：

```properties
## 采用db的存储形式
store.mode=db
## druid数据源
store.db.datasource=druid
## mysql数据库
store.db.dbType=mysql
## mysql驱动
store.db.driverClassName=com.mysql.jdbc.Driver
## TC的数据库url
store.db.url=jdbc:mysql://127.0.0.1:3306/seata_server?useUnicode=true
## 用户名
store.db.user=root
## 密码
store.db.password=Nov2014
```

在nacos中搜索上述的配置，直接修改其中的值，比如修改`store.mode`，如下图：

![](https://img.java-family.cn/seata/34.png)

当然Seata还支持Redis作为TC的数据库，只需要改动以下配置即可：

```properties
store.mode=redis
store.redis.host=127.0.0.1
store.redis.port=6379
store.redis.password=123456
```

### 启动TC

按照上述步骤全部配置成功后，则可以启动TC，在`seata-server-1.3.0\seata\bin`目录下直接点击`seata-server.bat`（windows）运行。

启动成功后，在Nacos的服务列表中则可以看到TC已经注册进入，如下图：

![](https://img.java-family.cn/seata/35.png)



至此，Seata的TC就启动完成了............



## Seata客户端搭建（RM）

上述已经将Seata的服务端（TC）搭建完成了，下面就以电商系统为例介绍一下如何编码实现分布式事务。

用户购买商品的业务逻辑。整个业务逻辑由3个微服务提供支持：

- 仓储服务：对给定的商品扣除仓储数量。
- 订单服务：根据采购需求创建订单。
- 帐户服务：从用户帐户中扣除余额。

需要了解的知识：Nacos和openFeign，有不清楚的可以看我的前两章教程，如下：

- [五十五张图告诉你微服务的灵魂摆渡者Nacos究竟有多强？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247493854&idx=1&sn=4b3fb7f7e17a76000733899f511ef915&scene=21#wechat_redirect)
- [openFeign夺命连环9问，这谁受得了？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247496653&idx=1&sn=7185077b3bdc1d094aef645d677ec472&scene=21#wechat_redirect)

### 仓储服务搭建

陈某整个教程使用的都是同一个聚合项目，关于Spring Cloud版本有不清楚的可以看我第一篇文章的说明。

#### 添加依赖

新建一个`seata-storage9020`项目，新增依赖如下：

![](https://img.java-family.cn/seata/36.png)

由于使用的`springCloud Alibaba`依赖版本是`2.2.1.RELEASE`，其中自带的seata版本是`1.1.0`，但是我们Seata服务端使用的版本是1.3.0，因此需要排除原有的依赖，重新添加1.3.0的依赖。

> 注意：seata客户端的依赖版本必须要和服务端一致。

#### 创建数据库

创建一个数据库`seata-storage`，其中新建两个表：

- `storage`：库存的业务表，SQL如下：

```sql
CREATE TABLE `storage`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `num` bigint(11) NULL DEFAULT NULL COMMENT '数量',
  `create_time` datetime(0) NULL DEFAULT NULL,
  `price` bigint(10) NULL DEFAULT NULL COMMENT '单价，单位分',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;

INSERT INTO `storage` VALUES (1, '码猿技术专栏', 1000, '2021-10-15 22:32:40', 100);
```

- **undo_log**：回滚日志表，这是Seata要求必须有的，每个业务库都应该创建一个，SQL如下：

```sql
CREATE TABLE `undo_log`  (
  `branch_id` bigint(20) NOT NULL COMMENT 'branch transaction id',
  `xid` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'global transaction id',
  `context` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` longblob NOT NULL COMMENT 'rollback info',
  `log_status` int(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` datetime(6) NOT NULL COMMENT 'create datetime',
  `log_modified` datetime(6) NOT NULL COMMENT 'modify datetime',
  UNIQUE INDEX `ux_undo_log`(`xid`, `branch_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci COMMENT = 'AT transaction mode undo table' ROW_FORMAT = Compact;
```

#### 配置seata相关配置

对于Nacos、Mysql数据源等相关信息就省略了，项目源码中都有。主要讲一下seata如何配置，详细配置如下：

```yaml
spring:
  application:
    ## 指定服务名称，在nacos中的名字
    name: seata-storage
## 客户端seata的相关配置
seata:
  ## 是否开启seata，默认true
  enabled: true
  application-id: ${spring.application.name}
  ## seata事务组的名称，一定要和config.tx(nacos)中配置的相同
  tx-service-group: ${spring.application.name}-tx-group
  ## 配置中心的配置
  config:
    ## 使用类型nacos
    type: nacos
    ## nacos作为配置中心的相关配置，需要和server在同一个注册中心下
    nacos:
      ## 命名空间，需要server端(registry和config)、nacos配置client端(registry和config)保持一致
      namespace: 7a7581ef-433d-46f3-93f9-5fdc18239c65
      ## 地址
      server-addr: localhost:8848
      ## 组， 需要server端(registry和config)、nacos配置client端(registry和config)保持一致
      group: SEATA_GROUP
      ## 用户名和密码
      username: nacos
      password: nacos
  registry:
    type: nacos
    nacos:
      ## 这里的名字一定要和seata服务端中的名称相同，默认是seata-server
      application: seata-server
      ## 需要server端(registry和config)、nacos配置client端(registry和config)保持一致
      group: SEATA_GROUP
      namespace: 7a7581ef-433d-46f3-93f9-5fdc18239c65
      username: nacos
      password: nacos
      server-addr: localhost:8848
```

以上配置注释已经很清楚，这里着重强调以下几点：

- 客户端seata中的nacos相关配置要和服务端相同，比如地址、命名空间..........
- tx-service-group：这个属性一定要注意，这个一定要和服务端的配置一致，否则不生效；比如上述配置中的，就要在nacos中新增一个配置`service.vgroupMapping.seata-storage-tx-group=default`，如下图：

![](https://img.java-family.cn/seata/37.png)

> 注意：`seata-storage-tx-group`仅仅是后缀，要记得添加配置的时候要加上前缀`service.vgroupMapping.`



#### 扣减库存的接口

逻辑很简单，这里仅仅是做了减库存的操作，代码如下：

![](https://img.java-family.cn/seata/38.png)

这里的接口并没有不同，还是使用`@Transactional`开启了本地事务，并没有涉及到分布式事务。

到这里仓储服务搭建好了..............

### 账户服务搭建

搭建完了仓储服务，账户服务搭建很类似了。

#### 添加依赖

新建一个`seata-account9021`服务，这里的依赖和仓储服务的依赖相同，直接复制

#### 创建数据库

创建一个`seata-account`数据库，其中新建了两个表：

- `account`：账户业务表，SQL如下：

```sql
CREATE TABLE `account`  (
  `id` bigint(11) NOT NULL,
  `user_id` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '用户userId',
  `money` bigint(11) NULL DEFAULT NULL COMMENT '余额，单位分',
  `create_time` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;

INSERT INTO `account` VALUES (1, 'abc123', 1000, '2021-10-19 17:49:53');
```

- **undo_log**：回滚日志表，同仓储服务



#### 配置seata相关配置

Seata相关配置和仓储服务相同，只不过需要在nacos中添加一个`service.vgroupMapping.seata-account-tx-group=default`，如下图：

![](https://img.java-family.cn/seata/39.png)



#### 扣减余额的接口

具体逻辑自己完善，这里我直接扣减余额，代码如下：

![](https://img.java-family.cn/seata/40.png)

依然没有涉及到分布式事务，还是使用`@Transactional`开启了本地事务，是不是很爽............



### 订单服务搭建（TM）

这里为了节省篇幅，陈某直接使用订单服务作为TM，下单、减库存、扣款整个流程都在订单服务中实现。

#### 添加依赖

新建一个`seata-order9022`服务，这里需要添加的依赖如下：

- Nacos服务发现的依赖
- seata的依赖
- openFeign的依赖，由于要调用账户、仓储的微服务，因此需要额外添加一个openFeign的依赖



#### 创建数据库

新建一个`seata_order`数据库，其中新建两个表，如下：

- `t_order`：订单的业务表

```sql
CREATE TABLE `t_order`  (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `product_id` bigint(11) NULL DEFAULT NULL COMMENT '商品Id',
  `num` bigint(11) NULL DEFAULT NULL COMMENT '数量',
  `user_id` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '用户唯一Id',
  `create_time` datetime(0) NULL DEFAULT NULL,
  `status` int(1) NULL DEFAULT NULL COMMENT '订单状态 1 未付款 2 已付款 3 已完成',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 7 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
```

- **undo_log**：回滚日志表，同仓储服务

#### 配置和seata相关配置

Seata相关配置和仓储服务相同，只不过需要在nacos中添加一个`service.vgroupMapping.seata-order-tx-group=default`，如下图：

![](https://img.java-family.cn/seata/42.png)



#### 扣减库存的接口

这里需要通过openFeign调用仓储服务的接口进行扣减库存，接口如下：

![](https://img.java-family.cn/seata/44.png)

以上只是简单的通过openFeign调用，更细致的配置，比如降级，自己完善.........

#### 扣减余额的接口

这里仍然是通过openFeign调用账户服务的接口进行扣减余额，接口如下：

![](https://img.java-family.cn/seata/43.png)

#### 创建订单的接口

下订单的接口就是一个事务发起方，作为TM，需要发起一个全局事务，详细代码如下图：

![](https://img.java-family.cn/seata/45.png)

有什么不同？不同之处就是使用了`@GlobalTransactional`而不是`@Transactional`。

`@GlobalTransactional`是Seata提供的，用于开启才能全局事务，只在TM中标注即可生效。



### 测试

分别启动`seata-account9021`、`seata-storage9020`、`seata-order9022`，如下图：

![](https://img.java-family.cn/seata/46.png)

下面调用下单接口，如下图：

![](https://img.java-family.cn/seata/47.png)

从控制台输出的日志可以看出，流程未出现任何异常，事务已经提交，如下图：

![](https://img.java-family.cn/seata/48.png)

果然，查看订单、余额、库存表，数据也都是正确的。

但是，这仅仅是流程没问题，并不能说明分布式事务已经配置成功了，因此需要手动造个异常。

在扣减余额的接口睡眠2秒钟，因为openFeign的超时时间默认是1秒，这样肯定是超时异常了，如下图：

![](https://img.java-family.cn/seata/49.png)

此时，调用创建订单的接口，控制台日志输出如下图：

![](https://img.java-family.cn/seata/50.png)

发现在扣减余额处理中超时了，导致了异常.......

此时，看下库存的数据有没有扣减，很高兴，库存没有扣减成功，说明事务已经回滚了，分布式事务成功了。

### 总结

Seata客户端创建很简单，需要注意以下几点内容：

- seata客户端的版本需要和服务端保持一致
- 每个服务的数据库都要创建一个`undo_log`回滚日志表
- 客户端指定的事务分组名称要和Nacos相同，比如`service.vgroupMapping.seata-account-tx-group=default`
  - 前缀：`service.vgroupMapping.`
  - 后缀：`{自定义}`

> 项目源码已经上传，关注公众号`码猿技术专栏`回复关键词`9528`获取！

## AT模式原理分析

AT模式最大的优点就是对业务代码无侵入，一切都像在写单体业务逻辑一样。

TC相关的三张表：

- `global_table`：全局事务表，每当有一个全局事务发起后，就会在该表中记录全局事务的ID
- `branch_table`：分支事务表，记录每一个分支事务的ID，分支事务操作的哪个数据库等信息
- `lock_table`：全局锁

### 一阶段步骤

1. `TM：seata-order.create()`方法执行时，由于该方法具有`@GlobalTranscational`标志，该TM会向TC发起全局事务，生成XID（全局锁）
2. `RM：StorageService.deduct()`：写表，UNDO_LOG记录回滚日志（Branch ID），通知TC操作结果
3. `RM：AccountService.deduct()`：写表，UNDO_LOG记录回滚日志（Branch ID），通知TC操作结果
4. `RM：OrderService.create()`：写表，UNDO_LOG记录回滚日志（Branch ID），通知TC操作结果

RM写表的过程，Seata 会拦截**业务SQL**，首先解析 SQL 语义，在业务数据被更新前，将其保存成**before image**（前置镜像），然后执行**业务SQL**，在业务数据更新之后，再将其保存成**after image**（后置镜像），最后生成行锁。以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性。

![](https://img.java-family.cn/seata/51.png)

### 二阶段步骤

因为“业务 SQL”在一阶段已经提交至数据库， 所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。

**正常**：TM执行成功，通知TC全局提交，TC此时通知所有的RM提交成功，删除UNDO_LOG回滚日志

![](https://img.java-family.cn/seata/52.png)

**异常**：TM执行失败，通知TC全局回滚，TC此时通知所有的RM进行回滚，根据UNDO_LOG反向操作，使用**before image**还原业务数据，删除UNDO_LOG，但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

![](https://img.java-family.cn/seata/53.png)

> AT 模式的一阶段、二阶段提交和回滚均由 Seata 框架自动生成，用户只需编写**业务 SQL**，便能轻松接入分布式事务，AT 模式是一种对业务无任何侵入的分布式事务解决方案。



## 总结

本文介绍了七种分布式事务解决方案，以及阿里开源的Seata，从入门到实现，文中如有错误之处，欢迎留言指正。

本文只介绍了Seata的AT模式，其实Seata还支持TCC、Saga事务模式，关于这一部分内容和Seata源码分析会在下期文章中介绍。

















